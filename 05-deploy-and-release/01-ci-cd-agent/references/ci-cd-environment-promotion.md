---
name: ci-cd-environment-promotion
description: This skill defines how the CI/CD Agent promotes a validated change between Snowflake environments, deploying the identical commit-pinned artifact from DEV to QA to PROD, refreshing test data with zero-copy cloning, and detecting schema drift through INFORMATION_SCHEMA comparisons before and after each move. Use this skill for any task involving promoting releases across environments, preparing a QA or PROD deployment, blocking promotion without approval, cloning production data for testing, comparing environments for drift, or rolling back a failed promotion.
---

# CI/CD Environment Promotion

## Overview

Promotion moves one immutable, commit-pinned artifact through an ordered set of environments: DEV, then QA, then PROD. DEV deployments run automatically after checks pass. QA and PROD are protected, so the agent prepares them (computes drift and dry-run readiness) and executes them only after a Release Gatekeeper approval.

## When to Use

- A run reaches STATUS = 'READY' and needs promotion preparation.
- A Gatekeeper approved a request and the agent must run the deployment.
- Environments have diverged and need a drift report.
- A promotion failed and needs a rollback.

## Instructions

1. Define promotion order and protection in configuration.

        CREATE OR ALTER TABLE CICD_DB.CONTROL.ENVIRONMENTS (
            ENV_NAME STRING, DATABASE_NAME STRING, PROMOTION_ORDER NUMBER, IS_PROTECTED BOOLEAN
        );

        INSERT INTO CICD_DB.CONTROL.ENVIRONMENTS VALUES
            ('DEV',  'SALES_DEV_DB',  1, FALSE),
            ('QA',   'SALES_QA_DB',   2, TRUE),
            ('PROD', 'SALES_PROD_DB', 3, TRUE);

2. Refresh QA test data with zero-copy clone, never by copying PROD rows.

        CREATE OR REPLACE SCHEMA SALES_QA_DB.CORE_SNAPSHOT CLONE SALES_PROD_DB.CORE;

    Apply masking policies on cloned sensitive columns before granting QA roles access.

3. Detect drift between the source and target environments before promotion.

        WITH src AS (
            SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE
            FROM SALES_QA_DB.INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA <> 'INFORMATION_SCHEMA'
        ), tgt AS (
            SELECT TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME, DATA_TYPE
            FROM SALES_PROD_DB.INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA <> 'INFORMATION_SCHEMA'
        )
        SELECT 'ONLY_IN_SOURCE' AS DRIFT_TYPE, * FROM (SELECT * FROM src MINUS SELECT * FROM tgt)
        UNION ALL
        SELECT 'ONLY_IN_TARGET', * FROM (SELECT * FROM tgt MINUS SELECT * FROM src);

    Rows of type ONLY_IN_SOURCE are expected changes. Rows of type ONLY_IN_TARGET are unmanaged drift. Record them as a BLOCKING DRIFT check unless the release explicitly reconciles them.

4. Prepare the promotion request instead of deploying to a protected environment.

        INSERT INTO CICD_DB.CONTROL.PROMOTION_REQUESTS (RUN_ID, SOURCE_ENV, TARGET_ENV, READINESS, DECISION)
            SELECT :run_id, 'QA', 'PROD', :readiness_json, 'PENDING';

5. Execute promotion only after approval, from the same commit.

        CREATE OR REPLACE PROCEDURE CICD_DB.ORCH.SP_EXECUTE_PROMOTION(P_REQUEST_ID STRING)
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS OWNER
        AS
        $$
        DECLARE
            v_commit STRING; v_target STRING; v_decision STRING; v_run STRING; v_path STRING;
        BEGIN
            SELECT r.COMMIT_HASH, p.TARGET_ENV, p.DECISION, p.RUN_ID
                INTO :v_commit, :v_target, :v_decision, :v_run
                FROM CICD_DB.CONTROL.PROMOTION_REQUESTS p
                JOIN CICD_DB.CONTROL.PIPELINE_RUNS r ON r.RUN_ID = p.RUN_ID
                WHERE p.REQUEST_ID = :P_REQUEST_ID;

            IF (v_decision <> 'APPROVED') THEN
                RETURN 'BLOCKED: decision is ' || v_decision;
            END IF;

            v_path := '@CICD_DB.SRC.DATA_REPO/commits/' || v_commit || '/deploy/deploy.sql';
            EXECUTE IMMEDIATE 'EXECUTE IMMEDIATE FROM ' || v_path || ' USING (env => ''SALES_' || v_target || ''')';

            UPDATE CICD_DB.CONTROL.PIPELINE_RUNS
                SET STATUS = 'PROMOTED', TARGET_ENV = :v_target, UPDATED_AT = CURRENT_TIMESTAMP()
                WHERE RUN_ID = :v_run;
            RETURN 'PROMOTED:' || v_target;
        END;
        $$;

6. Run post-promotion validation. Rerun the drift query and smoke checks against the target, and record the results under the same RUN_ID. Detailed validation belongs to the Deployment Validation Agent.

7. Roll back by promoting the previous PROMOTED commit through the same procedure. For data-destructive changes, take a pre-promotion clone first.

        CREATE OR REPLACE SCHEMA SALES_PROD_DB.CORE_PRE_<run_id_short> CLONE SALES_PROD_DB.CORE;

## Coding Conventions

- Promote one commit hash per request. Never mix commits in a single promotion.
- Only CICD_DEPLOYER executes promotions. RELEASE_GATEKEEPER decides but does not deploy.
- Never skip an environment in PROMOTION_ORDER.
- Pre-promotion clones use the suffix _PRE_<run_id_short> and are dropped by retention policy after a set number of days.
- Use Time Travel and clone for recovery. Do not use manual DDL hotfixes in PROD.