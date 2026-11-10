---
name: ci-cd-build-and-checks
description: This skill defines how the CI/CD Agent builds and validates a detected change, packaging Snowpark code with Snowflake CLI, rendering Jinja-templated SQL from the Git repository with EXECUTE IMMEDIATE FROM, and running blocking and warning checks recorded in the CHECK_RESULTS control table. Use this skill for any task involving snow snowpark build or deploy, compiling or dry-running SQL migrations, Snowpark Python unit tests with pytest, data metric function checks, policy and naming linting, failing a build on errors, or giving developers fast pipeline feedback.
---

# CI/CD Build and Checks

## Overview

This skill covers the Acts stage up to promotion readiness. It takes a DETECTED run, builds artifacts once, deploys them to DEV, and runs automated checks. Any failed BLOCKING check halts the run immediately and marks it FAILED. WARNING checks are recorded and passed to the Gatekeeper.

## When to Use

- Packaging Snowpark procedures, UDFs, or UDTFs for deployment.
- Applying versioned SQL migrations from the repository to an environment.
- Adding unit, data quality, or policy checks to a pipeline.
- Diagnosing why a build halted.

## Instructions

1. Structure the repository so builds are predictable.

        repo/
            snowflake.yml              # Snowflake CLI project definition
            app/                       # Snowpark Python source
            tests/                     # pytest unit tests
            deploy/
                deploy.sql             # entry point, Jinja-templated
                migrations/V001__init.sql
            checks/
                data_quality.sql
                policy.sql

2. Build and test Snowpark code in the CI runner, then upload the artifact keyed by commit.

        pip install -r requirements.txt
        pytest tests/ --junitxml=reports/unit.xml
        snow snowpark build
        snow stage copy app.zip @CICD_DB.ARTIFACTS.BUILD_STAGE/${COMMIT_SHA}/ --connection cicd
        snow snowpark deploy --replace --connection cicd_dev

3. Render and apply SQL from the repository for a target environment.

        -- deploy/deploy.sql
        CREATE OR ALTER SCHEMA {{ env }}_DB.CORE;
        CREATE OR ALTER TABLE {{ env }}_DB.CORE.ORDERS (
            ORDER_ID NUMBER, CUSTOMER_ID NUMBER, AMOUNT NUMBER(12,2), UPDATED_AT TIMESTAMP_LTZ
        );

        -- Executed by the agent:
        EXECUTE IMMEDIATE FROM @CICD_DB.SRC.DATA_REPO/branches/main/deploy/deploy.sql
            USING (env => 'SALES_DEV');

    To pin to the exact detected commit, use @CICD_DB.SRC.DATA_REPO/commits/<commit_hash>/deploy/deploy.sql.

4. Record every check result through one procedure.

        CREATE OR REPLACE PROCEDURE CICD_DB.ORCH.SP_RECORD_CHECK(
            P_RUN_ID STRING, P_NAME STRING, P_TYPE STRING, P_SEVERITY STRING, P_PASSED BOOLEAN, P_DETAIL VARIANT)
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS OWNER
        AS
        $$
        BEGIN
            INSERT INTO CICD_DB.CONTROL.CHECK_RESULTS (RUN_ID, CHECK_NAME, CHECK_TYPE, SEVERITY, PASSED, DETAIL)
                SELECT :P_RUN_ID, :P_NAME, :P_TYPE, :P_SEVERITY, :P_PASSED, :P_DETAIL;
            IF (NOT P_PASSED AND P_SEVERITY = 'BLOCKING') THEN
                UPDATE CICD_DB.CONTROL.PIPELINE_RUNS
                    SET STATUS = 'FAILED', FAILURE_STAGE = 'CHECK:' || :P_NAME,
                        FAILURE_DETAIL = :P_DETAIL, UPDATED_AT = CURRENT_TIMESTAMP()
                    WHERE RUN_ID = :P_RUN_ID;
                RETURN 'HALT';
            END IF;
            RETURN 'CONTINUE';
        END;
        $$;

5. Apply this minimum check set:

    | Check | Type | Severity |
    |---|---|---|
    | Unit tests pass | UNIT | BLOCKING |
    | All migrations apply cleanly in DEV | BUILD | BLOCKING |
    | Primary key uniqueness and not-null on key columns | DATA_QUALITY | BLOCKING |
    | Row count delta within tolerance | DATA_QUALITY | WARNING |
    | Naming conventions and no grants to PUBLIC | POLICY | BLOCKING |
    | Object comments present | POLICY | WARNING |

6. Data quality check example, using system data metric functions where available.

        SELECT SNOWFLAKE.CORE.DUPLICATE_COUNT(SELECT ORDER_ID FROM SALES_DEV_DB.CORE.ORDERS) AS DUPES,
               SNOWFLAKE.CORE.NULL_COUNT(SELECT ORDER_ID FROM SALES_DEV_DB.CORE.ORDERS) AS NULLS;

7. Halt on failure. Any procedure that receives 'HALT' raises an exception so the Task graph stops downstream tasks.

        LET r STRING := (CALL CICD_DB.ORCH.SP_RECORD_CHECK(:run_id, 'pk_unique', 'DATA_QUALITY', 'BLOCKING', :passed, :detail));
        IF (r = 'HALT') THEN
            LET e EXCEPTION (-20001, 'Blocking check failed');
            RAISE e;
        END IF;

8. When all BLOCKING checks pass, set STATUS = 'CHECKING' to 'READY' and hand off to ci-cd-environment-promotion.

## Coding Conventions

- Build exactly once per commit and store the artifact under that commit's hash.
- Migrations are forward-only and named V<NNN>__<description>.sql. Never edit an applied migration.
- Every check has a stable CHECK_NAME so results compare across runs.
- Check detail VARIANT payloads include expected value, actual value, and the query used.
- Keep the developer feedback loop short. Unit and policy checks run before any warehouse-heavy data checks.