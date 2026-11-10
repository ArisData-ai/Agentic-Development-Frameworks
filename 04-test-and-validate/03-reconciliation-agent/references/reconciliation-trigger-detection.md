---
name: reconciliation-trigger-detection
description: This skill defines how the Reconciliation Agent detects completed loads and migration cutover milestones, registers reconciliation configuration, captures source-side control evidence, and queues reconciliation runs automatically through Snowflake Streams, Tasks, and stored procedures. Use this skill for any task involving setting up RECON_CONFIG or LOAD_AUDIT tables, recording source row counts and control totals at extraction, creating streams and task chains that trigger reconciliation after Openflow, Snowpipe, COPY INTO, or CDC loads, or scheduling reconciliation for mock cutover, parallel run, and go-live milestones.
---

# Reconciliation Trigger Detection

## Overview
This skill implements the SENSE stage. It ensures that every completed load and every cutover milestone produces a queued reconciliation run without manual intervention, and that source-side evidence is captured at the moment of extraction.

## When to Use
- Initial setup of the reconciliation control framework.
- Onboarding a new source object or migration wave.
- Connecting reconciliation to Openflow, Snowpipe, COPY INTO, dbt, or custom ELT completion events.
- Scheduling milestone reconciliations (mock cutover, parallel run, final cutover).

## Instructions

1. Create the configuration table. One row per source-target pair:

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.RECON_CONFIG (
            CONFIG_ID               STRING DEFAULT UUID_STRING(),
            SOURCE_SYSTEM           STRING,          -- e.g. ORACLE_EBS, SAP_ECC, TERADATA_EDW
            SOURCE_OBJECT           STRING,          -- legacy object name
            SOURCE_SNAPSHOT_TABLE   STRING,          -- landed copy in Snowflake for field-level compare
            TARGET_OBJECT           STRING,          -- fully qualified Snowflake target
            KEY_COLUMNS             ARRAY,
            MEASURE_COLUMNS         ARRAY,           -- columns summed as control totals
            COMPARE_COLUMNS         ARRAY,           -- columns included in row hash
            FILTER_PREDICATE        STRING,          -- optional scope, e.g. FISCAL_PERIOD = '2026-10'
            ROW_COUNT_TOLERANCE_PCT NUMBER(9,4) DEFAULT 0,
            AMOUNT_TOLERANCE_ABS    NUMBER(38,4) DEFAULT 0,
            CRITICALITY             STRING,          -- HIGH, MEDIUM, LOW
            IS_ACTIVE               BOOLEAN DEFAULT TRUE,
            CONFIG_VERSION          NUMBER DEFAULT 1,
            APPROVED_BY             STRING,
            CREATED_AT              TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Create the load audit table. Loaders write one row per completed load, including source-side counts and totals computed in the source system at extraction:

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.LOAD_AUDIT (
            LOAD_ID                 STRING,
            SOURCE_SYSTEM           STRING,
            SOURCE_OBJECT           STRING,
            TARGET_OBJECT           STRING,
            LOAD_TYPE               STRING,          -- FULL, INCREMENTAL, CDC
            LOAD_STATUS             STRING,          -- COMPLETED, FAILED
            MILESTONE               STRING,          -- NULL, MOCK_CUTOVER_1, PARALLEL_RUN, GO_LIVE
            SOURCE_ROW_COUNT        NUMBER,
            SOURCE_CONTROL_TOTALS   VARIANT,         -- {"AMOUNT": 1250000.25, "QTY": 4410}
            EXTRACT_WATERMARK       STRING,
            COMPLETED_AT            TIMESTAMP_LTZ
        );

3. Create the run queue:

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.RECON_RUN (
            RUN_ID          STRING,
            LOAD_ID         STRING,
            CONFIG_ID       STRING,
            MILESTONE       STRING,
            TRIGGER_TYPE    STRING,                  -- LOAD, CUTOVER, MANUAL
            STATUS          STRING,                  -- QUEUED, RUNNING, PASS, WARN, FAIL, ERROR
            CREATED_AT      TIMESTAMP_LTZ,
            STARTED_AT      TIMESTAMP_LTZ,
            ENDED_AT        TIMESTAMP_LTZ
        );

4. Create the stream and enqueue procedure. The INSERT consumes the stream so each load is queued once:

        CREATE STREAM IF NOT EXISTS RECON_DB.CONTROL.LOAD_AUDIT_STRM
            ON TABLE RECON_DB.CONTROL.LOAD_AUDIT APPEND_ONLY = TRUE;

        CREATE OR REPLACE PROCEDURE RECON_DB.CONTROL.SP_ENQUEUE_RECON_RUNS()
        RETURNS STRING
        LANGUAGE SQL
        AS
        $$
        BEGIN
            INSERT INTO RECON_DB.CONTROL.RECON_RUN
                (RUN_ID, LOAD_ID, CONFIG_ID, MILESTONE, TRIGGER_TYPE, STATUS, CREATED_AT)
            SELECT UUID_STRING(), s.LOAD_ID, c.CONFIG_ID, s.MILESTONE,
                   IFF(s.MILESTONE IS NOT NULL, 'CUTOVER', 'LOAD'),
                   'QUEUED', CURRENT_TIMESTAMP()
            FROM RECON_DB.CONTROL.LOAD_AUDIT_STRM s
            JOIN RECON_DB.CONTROL.RECON_CONFIG c
              ON c.TARGET_OBJECT = s.TARGET_OBJECT
             AND c.IS_ACTIVE
            WHERE s.LOAD_STATUS = 'COMPLETED';
            RETURN 'ENQUEUED: ' || SQLROWCOUNT;
        END;
        $$;

5. Create the task chain. The root task fires only when new audit rows exist; the child task executes queued runs:

        CREATE OR REPLACE TASK RECON_DB.CONTROL.TSK_RECON_TRIGGER
            WAREHOUSE = RECON_WH
            SCHEDULE  = '5 MINUTE'
            WHEN SYSTEM$STREAM_HAS_DATA('RECON_DB.CONTROL.LOAD_AUDIT_STRM')
        AS
            CALL RECON_DB.CONTROL.SP_ENQUEUE_RECON_RUNS();

        CREATE OR REPLACE TASK RECON_DB.CONTROL.TSK_RECON_EXECUTE
            WAREHOUSE = RECON_WH
            AFTER RECON_DB.CONTROL.TSK_RECON_TRIGGER
        AS
            CALL RECON_DB.CONTROL.SP_RUN_QUEUED_RECONS();

        ALTER TASK RECON_DB.CONTROL.TSK_RECON_EXECUTE RESUME;
        ALTER TASK RECON_DB.CONTROL.TSK_RECON_TRIGGER RESUME;

6. Milestone runs. For a cutover milestone, insert one LOAD_AUDIT row per in-scope object with the MILESTONE value set, or enqueue directly with TRIGGER_TYPE = 'MANUAL' for ad hoc re-runs requested by the QA Lead.

7. Failed loads. Rows with LOAD_STATUS = 'FAILED' are not reconciled; surface them in the report as NOT_RECONCILED so gaps are visible rather than silent.

## Coding Conventions
- Source control totals must be computed in the source system, never derived from the Snowflake target.
- Resume child tasks before the root task; suspend the root task before altering the chain.
- Keep the root task schedule at 5 minutes or longer; the WHEN clause prevents warehouse spend when no loads complete.
- Store control totals as VARIANT keyed by MEASURE_COLUMNS names so the comparison engine can map them generically.
- Config changes create a new row with CONFIG_VERSION incremented and APPROVED_BY populated; deactivate the old row instead of updating it.