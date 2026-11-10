---
name: dq-check-execution
description: This skill defines how the Data Quality Agent evaluates freshness, completeness, uniqueness, validity, and distribution rules against monitored tables, attaching system and custom data metric functions and writing normalized pass or fail results per rule and run, built on Snowflake Horizon data metric functions, Snowpark, and Tasks. Use this skill for any task involving running data quality checks, creating custom DMFs, setting DMF schedules, null count or duplicate checks, referential and accepted value validation, threshold evaluation, or reporting pass rates for finance, retail, and ERP tables.
---
# DQ Check Execution

## Overview
Check execution turns ACTIVE rules in DQ_DB.CONFIG.DQ_RULES into measurements. Horizon data metric functions (DMFs) compute metrics; the agent compares each metric to its threshold and writes a normalized result that drives quarantine and escalation.

## When to Use
- Attaching quality metrics to new or existing tables
- Writing custom checks not covered by system DMFs
- Evaluating rules for an open run
- Producing pass rate evidence for audits and QA review

## Instructions

### 1. Map dimensions to metric functions
    FRESHNESS     -> SNOWFLAKE.CORE.FRESHNESS(ts_col)          threshold: seconds <= SLA
    COMPLETENESS  -> SNOWFLAKE.CORE.NULL_COUNT(col)            threshold: <= allowed nulls
                     SNOWFLAKE.CORE.NULL_PERCENT(col)          threshold: <= allowed percent
    UNIQUENESS    -> SNOWFLAKE.CORE.DUPLICATE_COUNT(col)       threshold: = 0
    VOLUME        -> SNOWFLAKE.CORE.ROW_COUNT()                threshold: BETWEEN low AND high
    VALIDITY      -> custom DMF (accepted values, ranges, referential)
    DISTRIBUTION  -> custom DMF or SP comparing to baseline profile

### 2. Attach DMFs and schedule on change
    ALTER TABLE RAW_DB.ERP.SALES_ORDERS SET DATA_METRIC_SCHEDULE = 'TRIGGER_ON_CHANGES';

    ALTER TABLE RAW_DB.ERP.SALES_ORDERS
        ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.NULL_COUNT ON (CUSTOMER_ID);
    ALTER TABLE RAW_DB.ERP.SALES_ORDERS
        ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.DUPLICATE_COUNT ON (ORDER_ID);
    ALTER TABLE RAW_DB.ERP.SALES_ORDERS
        ADD DATA METRIC FUNCTION SNOWFLAKE.CORE.FRESHNESS ON (LOADED_AT);

Where the account supports DMF expectations, register the threshold alongside the metric so Horizon reports violations natively, and keep DQ_RULES as the system of record:

    -- ALTER TABLE ... MODIFY DATA METRIC FUNCTION SNOWFLAKE.CORE.NULL_COUNT ON (CUSTOMER_ID)
    --     ADD EXPECTATION NO_NULL_CUSTOMER (VALUE = 0);

### 3. Create custom DMFs for validity and distribution
    CREATE OR REPLACE DATA METRIC FUNCTION DQ_DB.OPS.DMF_INVALID_STATUS_COUNT(
        ARG_T TABLE (ARG_C STRING))
    RETURNS NUMBER
    AS
    $$
        SELECT COUNT_IF(ARG_C NOT IN ('OPEN','SHIPPED','INVOICED','CLOSED','CANCELLED')) FROM ARG_T
    $$;

    CREATE OR REPLACE DATA METRIC FUNCTION DQ_DB.OPS.DMF_NEGATIVE_AMOUNT_COUNT(
        ARG_T TABLE (ARG_C NUMBER))
    RETURNS NUMBER
    AS
    $$
        SELECT COUNT_IF(ARG_C < 0) FROM ARG_T
    $$;

For distribution, store a baseline profile per column (mean, stddev, p05, p95, distinct ratio, top category shares) in DQ_DB.CONFIG.COLUMN_BASELINES and compute drift in a Snowpark procedure; flag when mean shifts beyond 3 baseline stddev or a category share moves more than 10 percentage points.

### 4. Evaluate rules for a run
    CREATE TABLE IF NOT EXISTS DQ_DB.RESULTS.DQ_CHECK_RESULTS (
        RESULT_ID       STRING DEFAULT UUID_STRING(),
        RUN_ID          STRING,
        RULE_ID         STRING,
        TABLE_FQN       STRING,
        COLUMN_NAME     STRING,
        DIMENSION       STRING,
        METRIC_VALUE    FLOAT,
        THRESHOLD_LOW   FLOAT,
        THRESHOLD_HIGH  FLOAT,
        PASSED          BOOLEAN,
        SEVERITY        STRING,
        FAILED_ROWS     NUMBER,
        MEASURED_AT     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

SP_RUN_QUALITY_CHECKS(RUN_ID) must:
1. Read ACTIVE rules for the run's table from DQ_RULES.
2. Compute each metric on demand for the run's scope (SELECT SNOWFLAKE.CORE.NULL_COUNT(SELECT col FROM table WHERE load filter)) or read the latest value from SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS.
3. Apply OPERATOR and thresholds to set PASSED.
4. Insert one DQ_CHECK_RESULTS row per rule, then set DQ_RUN_LOG.STATUS = 'CHECKED'.

    -- On-demand evaluation example
    SELECT SNOWFLAKE.CORE.NULL_COUNT(
        SELECT CUSTOMER_ID FROM RAW_DB.ERP.SALES_ORDERS WHERE LOADED_AT >= :run_start);

### 5. Summarize outcomes
    SELECT RUN_ID, TABLE_FQN,
           COUNT(*)                         AS RULES_EVALUATED,
           COUNT_IF(PASSED)                 AS RULES_PASSED,
           ROUND(100 * COUNT_IF(PASSED) / COUNT(*), 2) AS PASS_RATE_PCT,
           COUNT_IF(NOT PASSED AND SEVERITY = 'BLOCKER') AS BLOCKERS
    FROM DQ_DB.RESULTS.DQ_CHECK_RESULTS
    WHERE RUN_ID = :run_id
    GROUP BY RUN_ID, TABLE_FQN;

## Coding Conventions
- Prefer system DMFs; write a custom DMF only when no system metric fits, and keep it to a single scalar output.
- Custom DMF signatures use ARG_T for the table and ARG_C, ARG_C2 for columns.
- Use TRIGGER_ON_CHANGES for load-driven tables and a CRON schedule for slowly changing or reference tables.
- Each check scopes to the current load where possible to control credit spend.
- Never mutate monitored data inside a check; checks are read-only.
- Record FAILED_ROWS whenever a row-level count is available so quarantine can verify its counts.