---
name: deployment-validation-agent
description: This skill defines the end-to-end Deployment Validation Agent that detects completed Snowflake deployments, runs post-deployment checks on objects, permissions, data flows, and performance through Snowpark procedures, and returns a go or no-go verdict with evidence to the Release Gatekeeper. Use this skill for any task involving post-deployment validation, release verification, go-live smoke testing, catching failed or partial deployments, setting up the validation framework schema, or wiring Streams, Tasks, Snowpark, and Cortex Agents into a release pipeline driven by GitHub Actions, Azure DevOps, Jenkins, schemachange, dbt, or Terraform.
---
# Deployment Validation Agent

## Overview
The Deployment Validation Agent confirms that each deployment landed correctly and works in the target environment within minutes of go-live. It runs a closed loop:

- **Sense:** Detects completed deployments recorded by the CI/CD pipeline.
- **Act:** Runs post-deployment checks across four categories (objects, permissions, data flows, performance) and flags any deviation from the expected state.
- **Hand off:** Sends a structured verdict and evidence package to the Release Gatekeeper, who confirms the release or triggers rollback.

The agent never rolls back or changes production objects itself. Its only output is evidence and a recommendation.

Target customers are teams with business-critical data products, where a failed release has immediate revenue or reporting impact.

## When to Use
- Building or extending a post-deployment validation framework in Snowflake.
- Adding automated go/no-go evidence to an existing CI/CD release process.
- Detecting partial deployments: missing objects, drifted grants, suspended tasks, stale streams, or performance regressions.
- Setting up the shared schema, roles, and warehouse that the sibling skills depend on.

## Instructions

### 1. Provision the framework foundation
    CREATE DATABASE IF NOT EXISTS DEPLOY_VALIDATION_DB;
    CREATE SCHEMA IF NOT EXISTS DEPLOY_VALIDATION_DB.CORE;

    CREATE WAREHOUSE IF NOT EXISTS VALIDATION_WH
        WAREHOUSE_SIZE = 'XSMALL'
        AUTO_SUSPEND = 60
        AUTO_RESUME = TRUE
        INITIALLY_SUSPENDED = TRUE;

    CREATE ROLE IF NOT EXISTS DEPLOY_VALIDATOR_ROLE;
    GRANT USAGE ON WAREHOUSE VALIDATION_WH TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT ALL ON DATABASE DEPLOY_VALIDATION_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT ALL ON SCHEMA DEPLOY_VALIDATION_DB.CORE TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT EXECUTE TASK ON ACCOUNT TO ROLE DEPLOY_VALIDATOR_ROLE;

### 2. Create the core control tables
    USE SCHEMA DEPLOY_VALIDATION_DB.CORE;

    -- Written by the CI/CD pipeline at the end of every deployment
    CREATE TABLE IF NOT EXISTS DEPLOYMENT_LOG (
        DEPLOYMENT_ID     STRING        NOT NULL,
        RELEASE_ID        STRING        NOT NULL,
        ENVIRONMENT       STRING        NOT NULL,
        TARGET_DATABASE   STRING        NOT NULL,
        STATUS            STRING        NOT NULL,   -- STARTED | COMPLETED | FAILED
        PIPELINE_TOOL     STRING,
        COMMIT_SHA        STRING,
        MANIFEST          VARIANT,                  -- expected objects, grants, flows, baselines
        COMPLETED_AT      TIMESTAMP_LTZ,
        CONSTRAINT PK_DEPLOYMENT_LOG PRIMARY KEY (DEPLOYMENT_ID)
    );

    CREATE TABLE IF NOT EXISTS VALIDATION_QUEUE (
        DEPLOYMENT_ID     STRING,
        ENQUEUED_AT       TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        PROCESSED_AT      TIMESTAMP_LTZ,
        RUN_ID            STRING
    );

    CREATE TABLE IF NOT EXISTS VALIDATION_RESULTS (
        RUN_ID            STRING,
        DEPLOYMENT_ID     STRING,
        CHECK_CATEGORY    STRING,      -- OBJECT | PERMISSION | DATA_FLOW | PERFORMANCE
        CHECK_NAME        STRING,
        OBJECT_NAME       STRING,
        SEVERITY          STRING,      -- CRITICAL | HIGH | MEDIUM | LOW
        STATUS            STRING,      -- PASS | WARN | FAIL | ERROR
        EXPECTED          VARIANT,
        ACTUAL            VARIANT,
        DETAILS           STRING,
        CHECKED_AT        TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE TABLE IF NOT EXISTS VALIDATION_VERDICTS (
        RUN_ID            STRING,
        DEPLOYMENT_ID     STRING,
        RELEASE_ID        STRING,
        ENVIRONMENT       STRING,
        VERDICT           STRING,      -- GO | GO_WITH_WARNINGS | NO_GO
        CRITICAL_FAILS    NUMBER,
        HIGH_FAILS        NUMBER,
        WARNINGS          NUMBER,
        SUMMARY           STRING,
        EVIDENCE          VARIANT,
        VALIDATION_SECS   NUMBER,
        CREATED_AT        TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 3. Define the deployment manifest contract
The CI/CD pipeline writes the manifest into `DEPLOYMENT_LOG.MANIFEST`. Every check reads its expected state from the manifest.

    {
      "objects": [
        {"type": "TABLE", "name": "SALES_DB.CORE.ORDERS",
         "columns": [{"name": "ORDER_ID", "type": "NUMBER"}, {"name": "ORDER_TS", "type": "TIMESTAMP_NTZ"}]},
        {"type": "VIEW", "name": "SALES_DB.MART.V_DAILY_REVENUE"},
        {"type": "TASK", "name": "SALES_DB.CORE.T_LOAD_ORDERS", "state": "started"}
      ],
      "grants": [
        {"role": "ANALYST_ROLE", "privilege": "SELECT", "on": "VIEW", "name": "SALES_DB.MART.V_DAILY_REVENUE"}
      ],
      "data_flows": [
        {"type": "TASK", "name": "SALES_DB.CORE.T_LOAD_ORDERS", "max_minutes_since_success": 30},
        {"type": "DYNAMIC_TABLE", "name": "SALES_DB.MART.DT_REVENUE", "max_lag_minutes": 15},
        {"type": "FRESHNESS", "name": "SALES_DB.CORE.ORDERS", "ts_column": "ORDER_TS", "max_minutes": 60}
      ],
      "performance": [
        {"name": "daily_revenue_smoke", "sql": "SELECT COUNT(*) FROM SALES_DB.MART.V_DAILY_REVENUE", "baseline_ms": 1200, "tolerance_pct": 50}
      ]
    }

### 4. Run the validation workflow
1. **Detect:** A stream on `DEPLOYMENT_LOG` feeds a triggered task that queues `COMPLETED` deployments (see `deployment-validation-triggers`).
2. **Validate objects and permissions:** Compare the actual catalog state and grants with the manifest (see `deployment-validation-object-permission-checks`).
3. **Validate data flows and performance:** Check task runs, stream staleness, dynamic table refreshes, pipe status, data freshness, and query performance against baselines (see `deployment-validation-data-flow-performance-checks`).
4. **Decide and escalate:** Combine the results into a verdict, have a Cortex Agent write the explanation, and notify the Release Gatekeeper (see `deployment-validation-state-comparison-escalation`).

### 5. Orchestrator procedure
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.RUN_DEPLOYMENT_VALIDATION(P_DEPLOYMENT_ID STRING)
    RETURNS VARIANT
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    DECLARE
        v_run_id STRING DEFAULT UUID_STRING();
        v_start  TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP();
        v_result VARIANT;
    BEGIN
        CALL DEPLOY_VALIDATION_DB.CORE.VALIDATE_OBJECTS(:P_DEPLOYMENT_ID, :v_run_id);
        CALL DEPLOY_VALIDATION_DB.CORE.VALIDATE_PERMISSIONS(:P_DEPLOYMENT_ID, :v_run_id);
        CALL DEPLOY_VALIDATION_DB.CORE.VALIDATE_DATA_FLOWS(:P_DEPLOYMENT_ID, :v_run_id);
        CALL DEPLOY_VALIDATION_DB.CORE.VALIDATE_PERFORMANCE(:P_DEPLOYMENT_ID, :v_run_id);
        CALL DEPLOY_VALIDATION_DB.CORE.BUILD_VERDICT_AND_ESCALATE(:P_DEPLOYMENT_ID, :v_run_id, :v_start)
            INTO :v_result;
        UPDATE DEPLOY_VALIDATION_DB.CORE.VALIDATION_QUEUE
            SET PROCESSED_AT = CURRENT_TIMESTAMP(), RUN_ID = :v_run_id
            WHERE DEPLOYMENT_ID = :P_DEPLOYMENT_ID AND PROCESSED_AT IS NULL;
        RETURN v_result;
    EXCEPTION
        WHEN OTHER THEN
            INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, SEVERITY, STATUS, DETAILS)
            VALUES (:v_run_id, :P_DEPLOYMENT_ID, 'FRAMEWORK', 'ORCHESTRATOR', 'CRITICAL', 'ERROR', :SQLERRM);
            RETURN OBJECT_CONSTRUCT('run_id', v_run_id, 'verdict', 'NO_GO', 'error', SQLERRM);
    END;
    $$;

## Best Practices
- Treat a validation that cannot finish (status `ERROR`) as `NO_GO`, never as a pass.
- Keep the whole run under 10 minutes so findings arrive within minutes of go-live.
- Version the manifest with the release code so the expected state always matches the commit.
- Validate non-production environments with the same framework so checks are proven before production.
- Keep the agent read-only on target databases; only the Release Gatekeeper decides on rollback.

## Coding Conventions
- Framework objects live in `DEPLOY_VALIDATION_DB.CORE`. Use uppercase names with prefixes: `T_` for tasks, `S_` for streams, `V_` for views, `SV_` for semantic views.
- Use fully qualified object names in every procedure and task.
- Every result row carries `RUN_ID`, `DEPLOYMENT_ID`, `CHECK_CATEGORY`, `SEVERITY`, and `STATUS`.
- Status values are limited to `PASS`, `WARN`, `FAIL`, `ERROR`. Severity values are limited to `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`.
- Store expected and actual values as `VARIANT` so evidence stays machine-readable.

## References
- `references/deployment-validation-triggers.md`: Detecting completed deployments with Streams and triggered Tasks, the CI/CD write-back contract, and queue handling.
- `references/deployment-validation-object-permission-checks.md`: Snowpark checks for object existence, column structure, object state, and grant drift against the manifest.
- `references/deployment-validation-data-flow-performance-checks.md`: Checks for task runs, stream staleness, dynamic table refreshes, Snowpipe status, data freshness, and query performance against baselines.
- `references/deployment-validation-state-comparison-escalation.md`: Verdict scoring, Cortex Agent comparison of actual and expected state, and the evidence handoff and notification to the Release Gatekeeper.