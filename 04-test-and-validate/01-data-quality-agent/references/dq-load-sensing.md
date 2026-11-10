---
name: dq-load-sensing
description: This skill defines how the Data Quality Agent detects new loads and shifts in freshness and volume on monitored tables, producing a run log entry that triggers downstream checks, built on Snowflake Streams, triggered Tasks, and Horizon freshness and row count metrics. Use this skill for any task involving load detection, change data capture for quality checks, freshness SLA monitoring, stale table alerts, volume drop or spike detection, or triggering DQ runs after ERP, CRM, Openflow, or batch ingestion completes.
---
# DQ Load Sensing

## Overview
Load sensing is the entry point of the Data Quality Agent. It notices when new data lands, when a table goes stale against its SLA, and when load volume deviates from normal, then opens a run that the check, quarantine, and escalation phases consume.

## When to Use
- Triggering quality checks automatically after each load
- Monitoring freshness SLAs on critical tables
- Detecting unexpected volume drops or spikes
- Onboarding a new table into event-driven monitoring

## Instructions

### 1. Create the run log
    CREATE TABLE IF NOT EXISTS DQ_DB.RESULTS.DQ_RUN_LOG (
        RUN_ID          STRING DEFAULT UUID_STRING(),
        TABLE_FQN       STRING,
        TRIGGER_TYPE    STRING,          -- NEW_LOAD | FRESHNESS_BREACH | VOLUME_ANOMALY | MANUAL
        ROWS_DETECTED   NUMBER,
        STATUS          STRING DEFAULT 'OPEN',  -- OPEN | CHECKED | ESCALATED | CLOSED
        DETAIL          VARIANT,
        CREATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        UPDATED_AT      TIMESTAMP_LTZ
    );

### 2. Attach a stream per monitored table
    CREATE STREAM IF NOT EXISTS DQ_DB.OPS.STR_SALES_ORDERS
        ON TABLE RAW_DB.ERP.SALES_ORDERS
        APPEND_ONLY = TRUE
        SHOW_INITIAL_ROWS = FALSE;

Use APPEND_ONLY for insert-only loads; use standard streams when updates must also be checked.

### 3. Trigger a run when data arrives
    CREATE OR REPLACE TASK DQ_DB.OPS.TSK_SENSE_SALES_ORDERS
        WAREHOUSE = DQ_SVC_WH
        WHEN SYSTEM$STREAM_HAS_DATA('DQ_DB.OPS.STR_SALES_ORDERS')
    AS
        CALL DQ_DB.OPS.SP_OPEN_RUN('RAW_DB.ERP.SALES_ORDERS', 'DQ_DB.OPS.STR_SALES_ORDERS');

    ALTER TASK DQ_DB.OPS.TSK_SENSE_SALES_ORDERS RESUME;

SP_OPEN_RUN must consume the stream (INSERT ... SELECT from it into a staging snapshot) so the offset advances, insert a DQ_RUN_LOG row with TRIGGER_TYPE = 'NEW_LOAD', then call SP_RUN_QUALITY_CHECKS with the new RUN_ID.

### 4. Monitor freshness on a schedule
    CREATE OR REPLACE TASK DQ_DB.OPS.TSK_FRESHNESS_SWEEP
        WAREHOUSE = DQ_SVC_WH
        SCHEDULE = 'USING CRON */30 * * * * UTC'
    AS
        INSERT INTO DQ_DB.RESULTS.DQ_RUN_LOG (TABLE_FQN, TRIGGER_TYPE, DETAIL)
        SELECT r.TABLE_FQN, 'FRESHNESS_BREACH',
               OBJECT_CONSTRUCT('minutes_stale', m.VALUE / 60, 'sla_min', t.FRESHNESS_SLA_MIN)
        FROM SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS m
        JOIN DQ_DB.CONFIG.MONITORED_TABLES t
          ON t.TABLE_FQN = m.TABLE_DATABASE || '.' || m.TABLE_SCHEMA || '.' || m.TABLE_NAME
        WHERE m.METRIC_NAME = 'FRESHNESS'
          AND m.MEASUREMENT_TIME = (SELECT MAX(MEASUREMENT_TIME) FROM SNOWFLAKE.LOCAL.DATA_QUALITY_MONITORING_RESULTS x
                                     WHERE x.TABLE_NAME = m.TABLE_NAME AND x.METRIC_NAME = 'FRESHNESS')
          AND m.VALUE / 60 > t.FRESHNESS_SLA_MIN;

### 5. Detect volume anomalies
Compare the current load row count against a trailing 14-load baseline; flag loads outside mean plus or minus 3 standard deviations, or a 50 percent drop for low-history tables.

    WITH hist AS (
        SELECT TABLE_FQN, AVG(ROWS_DETECTED) AS mu, STDDEV(ROWS_DETECTED) AS sigma
        FROM (SELECT TABLE_FQN, ROWS_DETECTED,
                     ROW_NUMBER() OVER (PARTITION BY TABLE_FQN ORDER BY CREATED_AT DESC) AS rn
              FROM DQ_DB.RESULTS.DQ_RUN_LOG WHERE TRIGGER_TYPE = 'NEW_LOAD')
        WHERE rn BETWEEN 2 AND 15
        GROUP BY TABLE_FQN
    )
    SELECT l.RUN_ID, l.TABLE_FQN, l.ROWS_DETECTED, h.mu, h.sigma
    FROM DQ_DB.RESULTS.DQ_RUN_LOG l JOIN hist h USING (TABLE_FQN)
    WHERE l.STATUS = 'OPEN'
      AND ABS(l.ROWS_DETECTED - h.mu) > 3 * NULLIF(h.sigma, 0);

Record anomalies in DETAIL and carry them into the escalation package; do not block the run on volume alone unless the rule severity is BLOCKER.

## Coding Conventions
- One stream and one sensing task per monitored table; name them STR_<TABLE> and TSK_SENSE_<TABLE>.
- Always consume the stream inside the triggered task to advance its offset.
- Use a dedicated XS warehouse (DQ_SVC_WH) with AUTO_SUSPEND = 60 for sensing tasks, or serverless tasks where cost profile favors them.
- Store all thresholds (SLA minutes, sigma multiplier) in DQ_DB.CONFIG, not inline.
- Keep sensing idempotent; a rerun for the same stream offset must not create duplicate runs.