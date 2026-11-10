---
name: testing-agent
description: This skill defines the end-to-end operating model for a Snowflake Testing Agent that detects new or changed code, pipelines, and models, generates edge-case and regression tests, runs them in Snowpark, and sends defect reports with likely causes to a QA Lead, with Cortex Agents handling orchestration and Streams and Tasks handling scheduling. Use this skill for any task involving setting up the Testing Agent, designing its sense, act, and hand-off loop, provisioning its database, roles, and warehouse, wiring the task graph, defining the QA Lead approval gate, expanding test coverage for stored procedures, views, dynamic tables, dbt models, or Openflow pipelines, or deciding which sibling skill handles change detection, test generation, test execution, or defect reporting.
---
# Testing Agent

## Overview
The Testing Agent expands test coverage without adding manual effort. It runs a continuous three-phase loop inside Snowflake:

- **Sense:** Detects new or changed code, pipelines, and models.
- **Act:** Generates edge-case and regression tests, runs them, and logs defects with likely causes.
- **Hand off:** Sends results and defect reports to the **QA Lead**, who accepts, reprioritizes, or rejects each finding.

The agent never changes production objects, never closes defects, and never promotes generated tests into the permanent regression suite without QA Lead approval.

Primary Snowflake components:

- **Cortex Agents:** Plan which test suites to run for each change and coordinate the tools.
- **Snowpark:** Runs test logic, fixtures, and comparisons inside the platform.
- **Streams & Tasks:** Trigger the loop on change and on a schedule.
- **Cortex Analyst:** Summarizes pass rates and defect trends for QA and release owners.

## When to Use
- Standing up the Testing Agent in a new Snowflake account or client environment.
- Designing or changing the end-to-end detect, generate, run, and report workflow.
- Setting up the QA Lead approval gate and the agent's guardrails.
- Fitting the Testing Agent into Segment 4 (Test & Validate) next to the Data Quality and Reconciliation Agents.
- Finding the right sibling skill for a specific phase.

## Instructions

### 1. Provision foundation objects

    USE ROLE SYSADMIN;

    CREATE DATABASE IF NOT EXISTS TESTING_AGENT_DB;
    CREATE SCHEMA IF NOT EXISTS TESTING_AGENT_DB.CONFIG;
    CREATE SCHEMA IF NOT EXISTS TESTING_AGENT_DB.TESTS;
    CREATE SCHEMA IF NOT EXISTS TESTING_AGENT_DB.RESULTS;
    CREATE SCHEMA IF NOT EXISTS TESTING_AGENT_DB.DEFECTS;
    CREATE SCHEMA IF NOT EXISTS TESTING_AGENT_DB.ORCH;

    CREATE WAREHOUSE IF NOT EXISTS TESTING_AGENT_WH
        WAREHOUSE_SIZE = 'XSMALL'
        AUTO_SUSPEND = 60
        AUTO_RESUME = TRUE
        INITIALLY_SUSPENDED = TRUE
        STATEMENT_TIMEOUT_IN_SECONDS = 300;

    USE ROLE SECURITYADMIN;

    CREATE ROLE IF NOT EXISTS TESTING_AGENT_ROLE;
    CREATE ROLE IF NOT EXISTS QA_LEAD_ROLE;

    GRANT USAGE ON WAREHOUSE TESTING_AGENT_WH TO ROLE TESTING_AGENT_ROLE;
    GRANT ALL ON DATABASE TESTING_AGENT_DB TO ROLE TESTING_AGENT_ROLE;
    GRANT ALL ON ALL SCHEMAS IN DATABASE TESTING_AGENT_DB TO ROLE TESTING_AGENT_ROLE;
    GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE TESTING_AGENT_ROLE;
    GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE TESTING_AGENT_ROLE;
    GRANT EXECUTE TASK ON ACCOUNT TO ROLE TESTING_AGENT_ROLE;

    -- Read-only access to the objects under test; never grant write on targets
    GRANT USAGE ON DATABASE <TARGET_DB> TO ROLE TESTING_AGENT_ROLE;
    GRANT USAGE ON ALL SCHEMAS IN DATABASE <TARGET_DB> TO ROLE TESTING_AGENT_ROLE;
    GRANT SELECT ON ALL TABLES IN DATABASE <TARGET_DB> TO ROLE TESTING_AGENT_ROLE;
    GRANT SELECT ON ALL VIEWS IN DATABASE <TARGET_DB> TO ROLE TESTING_AGENT_ROLE;
    GRANT SELECT ON FUTURE TABLES IN DATABASE <TARGET_DB> TO ROLE TESTING_AGENT_ROLE;

    GRANT USAGE ON DATABASE TESTING_AGENT_DB TO ROLE QA_LEAD_ROLE;
    GRANT USAGE ON SCHEMA TESTING_AGENT_DB.DEFECTS TO ROLE QA_LEAD_ROLE;
    GRANT USAGE ON SCHEMA TESTING_AGENT_DB.RESULTS TO ROLE QA_LEAD_ROLE;

### 2. Run the agent loop

    SENSE    -> CHANGE_EVENTS (PENDING)            [testing-agent-change-detection]
    ACT      -> TEST_CASES (PROPOSED | ACTIVE)     [testing-agent-test-generation]
             -> TEST_RUNS / TEST_RESULTS           [testing-agent-test-execution]
             -> DEFECT_LOG (OPEN)                  [testing-agent-defect-reporting]
    HAND OFF -> V_QA_REVIEW_QUEUE -> QA Lead decision
                ACCEPTED | REPRIORITIZED | REJECTED

### 3. Wire the task graph
Each stored procedure checks for pending work itself, so child tasks need no WHEN clause.

    USE ROLE TESTING_AGENT_ROLE;
    USE SCHEMA TESTING_AGENT_DB.ORCH;

    CREATE OR REPLACE TASK T_00_DETECT_CHANGES
        WAREHOUSE = TESTING_AGENT_WH
        SCHEDULE = 'USING CRON */30 * * * * UTC'
        SUSPEND_TASK_AFTER_NUM_FAILURES = 3
        COMMENT = 'Testing Agent: sense phase'
    AS
        CALL TESTING_AGENT_DB.ORCH.DETECT_CHANGES();

    CREATE OR REPLACE TASK T_10_GENERATE_TESTS
        WAREHOUSE = TESTING_AGENT_WH
        AFTER T_00_DETECT_CHANGES
    AS
        CALL TESTING_AGENT_DB.ORCH.GENERATE_TESTS();

    CREATE OR REPLACE TASK T_20_RUN_TESTS
        WAREHOUSE = TESTING_AGENT_WH
        AFTER T_10_GENERATE_TESTS
    AS
        CALL TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE('CHANGED');

    CREATE OR REPLACE TASK T_30_REPORT_DEFECTS
        WAREHOUSE = TESTING_AGENT_WH
        AFTER T_20_RUN_TESTS
    AS
        CALL TESTING_AGENT_DB.ORCH.REPORT_DEFECTS();

    -- Nightly full regression, independent of change events
    CREATE OR REPLACE TASK T_90_NIGHTLY_REGRESSION
        WAREHOUSE = TESTING_AGENT_WH
        SCHEDULE = 'USING CRON 0 2 * * * UTC'
    AS
        CALL TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE('FULL_REGRESSION');

    SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('TESTING_AGENT_DB.ORCH.T_00_DETECT_CHANGES');
    ALTER TASK T_90_NIGHTLY_REGRESSION RESUME;

### 4. Register Cortex Agent tools
Expose each phase as a tool so the Cortex Agent can plan suites for a given change and answer ad hoc requests such as "test the changes to ORDERS_DT".

    tools:
        - name: detect_changes
          type: stored_procedure
          target: TESTING_AGENT_DB.ORCH.DETECT_CHANGES
          purpose: List new or changed objects and their downstream dependents
        - name: generate_tests
          type: stored_procedure
          target: TESTING_AGENT_DB.ORCH.GENERATE_TESTS
          purpose: Create edge-case and regression tests for pending change events
        - name: run_test_suite
          type: stored_procedure
          target: TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE
          args: [run_scope]   # CHANGED | FULL_REGRESSION | OBJECT:<fqn>
          purpose: Run tests and record results
        - name: report_defects
          type: stored_procedure
          target: TESTING_AGENT_DB.ORCH.REPORT_DEFECTS
          purpose: Log defects with likely causes and queue them for the QA Lead
        - name: test_metrics
          type: cortex_analyst
          semantic_model: '@TESTING_AGENT_DB.CONFIG.SEMANTIC_MODELS/testing_metrics.yaml'
          purpose: Answer pass-rate and defect-trend questions
    orchestration_rules:
        - Always run detect_changes before generate_tests
        - Never call any tool that writes to a target database
        - Stop and hand off to the QA Lead after report_defects

### 5. Enforce guardrails
- Grant the agent only read access to target objects. Run destructive fixtures only on zero-copy clones.
- Keep generated tests in PROPOSED status until the QA Lead approves them for the regression suite.
- Cap compute with a resource monitor on TESTING_AGENT_WH and set a statement timeout.
- Tag all agent queries with QUERY_TAG so costs can be attributed.
- Log every agent decision so the QA Lead and auditors can trace it.

### 6. Output expectations
Every run produces:
- A TEST_RUNS record with scope, trigger, counts, duration, and warehouse credits.
- One TEST_RESULTS row per test, with status PASS, FAIL, or ERROR and sample failing rows.
- Deduplicated DEFECT_LOG entries, each with severity, likely cause, confidence, and the linked change event.
- A refreshed V_QA_REVIEW_QUEUE and a notification to the QA Lead when new high-severity defects are open.

## Coding Conventions
- Use fully qualified object names everywhere: DATABASE.SCHEMA.OBJECT.
- Use uppercase snake case for object names. Prefix tasks with T_<order>_ and views with V_.
- Store all tunables (models, thresholds, schedules, scope) in TESTING_AGENT_DB.CONFIG tables. Do not hardcode them.
- Make procedures idempotent: use MERGE rather than INSERT for state tables, and keep reruns safe.
- Set QUERY_TAG to a JSON string: {"agent":"testing","phase":"<phase>","run_id":"<id>"}.
- Use status enums in uppercase: PENDING, PROCESSED, PROPOSED, ACTIVE, RETIRED, PASS, FAIL, ERROR, OPEN, ACCEPTED, REPRIORITIZED, REJECTED, RESOLVED.
- Add a COMMENT to every agent-owned object that names its owner and purpose.

## References
| Skill file | Covers |
|---|---|
| references/testing-agent-change-detection.md | Sense phase: monitored scope, DDL and definition change detection, CI/Git change registration, downstream impact using OBJECT_DEPENDENCIES, CHANGE_EVENTS state |
| references/testing-agent-test-generation.md | Act phase: test contract, edge-case category templates, Cortex-generated logic tests, regression baselines, SQL validation, TEST_CASES lifecycle |
| references/testing-agent-test-execution.md | Act phase: Snowpark test runner, clone-based fixtures, timeouts and retries, PASS/FAIL/ERROR classification, TEST_RUNS and TEST_RESULTS |
| references/testing-agent-defect-reporting.md | Hand-off phase: defect fingerprinting, likely-cause analysis, severity rules, QA Lead review queue and decisions, notifications, Cortex Analyst metrics |