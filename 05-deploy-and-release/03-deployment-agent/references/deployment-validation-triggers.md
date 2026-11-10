---
name: deployment-validation-triggers
description: This skill defines how the Deployment Validation Agent detects completed deployments by having CI/CD pipelines write to a deployment log table, capturing changes with a Snowflake Stream, and running validation within minutes through triggered Tasks and a validation queue. Use this skill for any task involving deployment event detection, post-release triggers, stream and task setup, CI/CD write-back from GitHub Actions, Azure DevOps, GitLab, Jenkins, schemachange, dbt, or Terraform, queue processing, retries, or troubleshooting validations that did not start.
---
# Deployment Validation Triggers

## Overview
This skill covers the agent's Sense step. The CI/CD pipeline records each deployment in `DEPLOYMENT_LOG`. A stream captures new and updated rows, and a triggered task moves `COMPLETED` deployments into `VALIDATION_QUEUE`. A child task then runs the validation orchestrator. Because the task fires when the stream has data, there is no polling interval to wait for.

## When to Use
- Connecting a CI/CD pipeline to the validation framework.
- Creating or changing the stream and task chain that starts validation.
- Re-running validation for a deployment by hand, or after a framework fix.
- Diagnosing validations that never started or started late.

## Instructions

### 1. CI/CD write-back (final pipeline step)
Run this as the last step of the deployment job, using a service user with INSERT rights on `DEPLOYMENT_LOG`.

    INSERT INTO DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG
        (DEPLOYMENT_ID, RELEASE_ID, ENVIRONMENT, TARGET_DATABASE, STATUS,
         PIPELINE_TOOL, COMMIT_SHA, MANIFEST, COMPLETED_AT)
    SELECT
        '${DEPLOYMENT_ID}', '${RELEASE_ID}', 'PROD', 'SALES_DB', 'COMPLETED',
        'GITHUB_ACTIONS', '${GITHUB_SHA}', PARSE_JSON('${MANIFEST_JSON}'), CURRENT_TIMESTAMP();

Example GitHub Actions step with the Snowflake CLI:

    - name: Register deployment for validation
      run: |
        snow sql -q "INSERT INTO DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG
          (DEPLOYMENT_ID, RELEASE_ID, ENVIRONMENT, TARGET_DATABASE, STATUS, PIPELINE_TOOL, COMMIT_SHA, MANIFEST, COMPLETED_AT)
          SELECT '${{ github.run_id }}', '${{ github.ref_name }}', 'PROD', 'SALES_DB', 'COMPLETED',
                 'GITHUB_ACTIONS', '${{ github.sha }}', PARSE_JSON(\$\$$(cat manifest.json)\$\$), CURRENT_TIMESTAMP()"
      env:
        SNOWFLAKE_CONNECTIONS_DEFAULT_AUTHENTICATOR: SNOWFLAKE_JWT

### 2. Stream on the deployment log
    CREATE OR REPLACE STREAM DEPLOY_VALIDATION_DB.CORE.S_DEPLOYMENT_LOG
        ON TABLE DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG
        SHOW_INITIAL_ROWS = FALSE;

### 3. Triggered task that enqueues completed deployments
    CREATE OR REPLACE TASK DEPLOY_VALIDATION_DB.CORE.T_ENQUEUE_DEPLOYMENTS
        WAREHOUSE = VALIDATION_WH
        WHEN SYSTEM$STREAM_HAS_DATA('DEPLOY_VALIDATION_DB.CORE.S_DEPLOYMENT_LOG')
    AS
        INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_QUEUE (DEPLOYMENT_ID)
        SELECT DEPLOYMENT_ID
        FROM DEPLOY_VALIDATION_DB.CORE.S_DEPLOYMENT_LOG
        WHERE METADATA$ACTION = 'INSERT'
          AND STATUS = 'COMPLETED';

### 4. Child task that processes the queue
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.PROCESS_VALIDATION_QUEUE()
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    DECLARE
        c CURSOR FOR
            SELECT DISTINCT DEPLOYMENT_ID
            FROM DEPLOY_VALIDATION_DB.CORE.VALIDATION_QUEUE
            WHERE PROCESSED_AT IS NULL
            ORDER BY DEPLOYMENT_ID;
        v_count NUMBER DEFAULT 0;
    BEGIN
        FOR r IN c DO
            LET v_id STRING := r.DEPLOYMENT_ID;
            CALL DEPLOY_VALIDATION_DB.CORE.RUN_DEPLOYMENT_VALIDATION(:v_id);
            v_count := v_count + 1;
        END FOR;
        RETURN 'Validated deployments: ' || v_count;
    END;
    $$;

    CREATE OR REPLACE TASK DEPLOY_VALIDATION_DB.CORE.T_RUN_VALIDATION
        WAREHOUSE = VALIDATION_WH
        AFTER DEPLOY_VALIDATION_DB.CORE.T_ENQUEUE_DEPLOYMENTS
    AS
        CALL DEPLOY_VALIDATION_DB.CORE.PROCESS_VALIDATION_QUEUE();

    ALTER TASK DEPLOY_VALIDATION_DB.CORE.T_RUN_VALIDATION RESUME;
    ALTER TASK DEPLOY_VALIDATION_DB.CORE.T_ENQUEUE_DEPLOYMENTS RESUME;

### 5. Safety-net sweep for stuck queue items
    CREATE OR REPLACE TASK DEPLOY_VALIDATION_DB.CORE.T_SWEEP_STUCK_VALIDATIONS
        WAREHOUSE = VALIDATION_WH
        SCHEDULE = 'USING CRON */15 * * * * UTC'
    AS
        CALL DEPLOY_VALIDATION_DB.CORE.PROCESS_VALIDATION_QUEUE();

    ALTER TASK DEPLOY_VALIDATION_DB.CORE.T_SWEEP_STUCK_VALIDATIONS RESUME;

### 6. Manual re-run
    INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_QUEUE (DEPLOYMENT_ID) VALUES ('<deployment_id>');
    EXECUTE TASK DEPLOY_VALIDATION_DB.CORE.T_SWEEP_STUCK_VALIDATIONS;

### 7. Trigger diagnostics
    SELECT NAME, STATE, SCHEDULED_TIME, COMPLETED_TIME, ERROR_MESSAGE
    FROM TABLE(DEPLOY_VALIDATION_DB.INFORMATION_SCHEMA.TASK_HISTORY(
        SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP())))
    WHERE NAME IN ('T_ENQUEUE_DEPLOYMENTS', 'T_RUN_VALIDATION', 'T_SWEEP_STUCK_VALIDATIONS')
    ORDER BY SCHEDULED_TIME DESC;

## Best Practices
- Only the INSERT that reads the stream advances its offset; keep that INSERT as the only stream consumer.
- Resume child tasks before the root task, and suspend the root task before changing any task in the chain.
- Use key-pair authentication for the CI/CD service user; never store passwords in pipeline variables.
- Keep `DATA_RETENTION_TIME_IN_DAYS` on `DEPLOYMENT_LOG` at least 1 day so the stream cannot go stale between deployments.
- Generate the manifest from the deployment tool's change set (schemachange scripts, dbt manifest.json, Terraform plan) instead of writing it by hand.

## Coding Conventions
- Streams start with `S_`; tasks start with `T_`; queue and log tables use singular nouns with an uppercase snake_case suffix.
- Tasks always name a warehouse or are explicitly serverless; never rely on defaults.
- Pipeline placeholders use the `${VARIABLE}` form and are replaced by the CI/CD tool, never by string concatenation inside Snowflake.
- Each task does one thing: either consume the stream or run validation.