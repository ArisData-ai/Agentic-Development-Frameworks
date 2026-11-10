---
name: ci-cd-orchestration-and-failure-handling
description: This skill defines how the CI/CD Agent orchestrates pipeline stages and handles failures, chaining detect, build, check, and prepare steps in a Snowflake Task graph with a finalizer task, automatic retries, and error notifications, while a Cortex Agent coordinates stages and triages failures through stored procedure tools. Use this skill for any task involving scheduling or chaining CI/CD tasks, stream-triggered tasks, halting pipelines on failure, retrying transient errors, alerting on failed deployments, building a Cortex Agent that runs or explains pipelines, or answering questions like why a build failed or what is pending approval.
---

# CI/CD Orchestration and Failure Handling

## Overview

Orchestration turns the stage procedures into one dependable flow. Snowflake Tasks provide deterministic execution and dependency order. A finalizer task always runs to record the outcome. A Cortex Agent sits on top as the coordination and triage layer: it calls the same procedures as tools, explains failures in plain language, and suggests next actions. It never bypasses the Gatekeeper.

## When to Use

- Wiring the CI/CD stages into an automated, scheduled, or event-driven graph.
- Configuring retries, auto-suspend, and failure alerts.
- Giving developers and DevOps a conversational interface to pipeline status and failure root cause.

## Instructions

1. Build the Task graph.

        CREATE OR ALTER TASK CICD_DB.ORCH.T_ROOT_DETECT
            WAREHOUSE = CICD_WH
            SCHEDULE = '5 MINUTE'
            SUSPEND_TASK_AFTER_NUM_FAILURES = 3
            TASK_AUTO_RETRY_ATTEMPTS = 2
            ERROR_INTEGRATION = CICD_ERROR_INT
            CONFIG = '{"branch": "main"}'
        AS
            CALL CICD_DB.ORCH.SP_DETECT_CHANGES('main');

        CREATE OR ALTER TASK CICD_DB.ORCH.T_BUILD
            WAREHOUSE = CICD_WH
            AFTER CICD_DB.ORCH.T_ROOT_DETECT
        AS
            CALL CICD_DB.ORCH.SP_BUILD_LATEST();

        CREATE OR ALTER TASK CICD_DB.ORCH.T_CHECKS
            WAREHOUSE = CICD_WH
            AFTER CICD_DB.ORCH.T_BUILD
        AS
            CALL CICD_DB.ORCH.SP_RUN_CHECKS_LATEST();

        CREATE OR ALTER TASK CICD_DB.ORCH.T_PREPARE_PROMOTION
            WAREHOUSE = CICD_WH
            AFTER CICD_DB.ORCH.T_CHECKS
        AS
            CALL CICD_DB.ORCH.SP_PREPARE_PROMOTION_LATEST();

        CREATE OR ALTER TASK CICD_DB.ORCH.T_FINALIZE
            WAREHOUSE = CICD_WH
            FINALIZE = CICD_DB.ORCH.T_ROOT_DETECT
        AS
            CALL CICD_DB.ORCH.SP_FINALIZE_RUN();

2. Short-circuit when nothing changed. The root task sets a return value and each child reads it.

        -- inside SP_DETECT_CHANGES, before RETURN:
        CALL SYSTEM$SET_RETURN_VALUE(:result);

        -- inside SP_BUILD_LATEST:
        LET upstream STRING := SYSTEM$GET_PREDECESSOR_RETURN_VALUE();
        IF (upstream = 'NO_CHANGE') THEN
            RETURN 'SKIPPED';
        END IF;

3. Trigger on artifact uploads instead of a schedule when the external CI runner owns the build.

        CREATE OR ALTER TASK CICD_DB.ORCH.T_ON_ARTIFACT
            WAREHOUSE = CICD_WH
            WHEN SYSTEM$STREAM_HAS_DATA('CICD_DB.ARTIFACTS.BUILD_STAGE_STREAM')
        AS
            CALL CICD_DB.ORCH.SP_REGISTER_ARTIFACTS();

4. Resume children before the root, then enable the graph.

        SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('CICD_DB.ORCH.T_ROOT_DETECT');

5. Finalizer responsibilities in SP_FINALIZE_RUN:
    - Read task outcomes from INFORMATION_SCHEMA.TASK_HISTORY for the current graph run.
    - Mark any run still in BUILDING or CHECKING as FAILED, with FAILURE_STAGE set to the failed task name.
    - Send a failure notification that includes RUN_ID, stage, and error message.

6. Classify failures before retrying.

    | Class | Examples | Action |
    |---|---|---|
    | Transient | Warehouse queue timeout, Git fetch network error | Automatic retry through TASK_AUTO_RETRY_ATTEMPTS |
    | Code | Compilation error, failed unit test | Halt and notify the commit author |
    | Data | Blocking data quality check failed | Halt and route to the Data Quality Agent |
    | Environment | Drift, missing grants, missing object | Halt and route to the platform owner |

7. Create the Cortex Agent for coordination and triage. Expose the stage procedures as tools and restrict the agent to read, explain, and prepare actions.

        CREATE OR REPLACE AGENT CICD_DB.ORCH.CICD_AGENT
            COMMENT = 'Coordinates CI/CD stages and triages failures'
            FROM SPECIFICATION
            $$
            models:
                orchestration: auto
            instructions:
                orchestration: >
                    You coordinate CI/CD runs. Report run status, explain failures from CHECK_RESULTS
                    and FAILURE_DETAIL, and recommend fixes. Never approve promotions and never deploy
                    to protected environments.
                response: Respond concisely with run_id, stage, cause, and next action.
            tools:
                - tool_spec:
                    type: generic
                    name: get_run_status
                    description: Returns status and failure detail for a pipeline run
                    input_schema:
                        type: object
                        properties:
                            run_id: { type: string }
                        required: [run_id]
                - tool_spec:
                    type: generic
                    name: rerun_checks
                    description: Re-runs checks for a run in DEV only
                    input_schema:
                        type: object
                        properties:
                            run_id: { type: string }
                        required: [run_id]
            tool_resources:
                get_run_status:
                    type: procedure
                    identifier: CICD_DB.ORCH.SP_GET_RUN_STATUS
                    execution_environment: { type: warehouse, warehouse: CICD_WH }
                rerun_checks:
                    type: procedure
                    identifier: CICD_DB.ORCH.SP_RERUN_CHECKS_DEV
                    execution_environment: { type: warehouse, warehouse: CICD_WH }
            $$;

    Check the specification keys against the Cortex Agents version available in the account, and adjust the model and tool resource fields if needed.

8. Monitor the graph.

        SELECT NAME, STATE, ERROR_MESSAGE, SCHEDULED_TIME, COMPLETED_TIME
        FROM TABLE(CICD_DB.INFORMATION_SCHEMA.TASK_HISTORY(
            SCHEDULED_TIME_RANGE_START => DATEADD('day', -1, CURRENT_TIMESTAMP())))
        WHERE DATABASE_NAME = 'CICD_DB'
        ORDER BY SCHEDULED_TIME DESC;

## Coding Conventions

- One root task per tracked branch. Suffix task names with the branch for non-main pipelines, for example T_ROOT_DETECT_RELEASE.
- Every task body is a single CALL to a stored procedure. Keep logic out of task definitions.
- Always define a finalizer task. Never rely on downstream tasks to report failures.
- Cortex Agent tools are least-privilege. Promotion execution and approval procedures are never exposed as agent tools.
- Set SUSPEND_TASK_AFTER_NUM_FAILURES on every root task so a broken pipeline does not loop silently.