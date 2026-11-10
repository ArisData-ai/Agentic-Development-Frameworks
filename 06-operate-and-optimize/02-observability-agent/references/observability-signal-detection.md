---
name: observability-signal-detection
description: This skill defines how the Observability Agent detects latency, failure, freshness and volume anomalies by reading Task history, Dynamic Table refresh history, Snowpipe copy history and query history, then writing normalized signal events to a central SIGNAL_EVENTS table with static SLO rules and SNOWFLAKE.ML.ANOMALY_DETECTION baselines. Use this skill for any task involving pipeline health checks, failed Task detection, Dynamic Table lag monitoring, stale data alerts, row count drops, slow query detection, anomaly baselines, or building the sensing layer of an observability solution.
---
# Observability Signal Detection

## Overview
This skill implements the Sense stage. It converts raw Snowflake telemetry into one normalized table of signals, each tagged with pipeline, signal type, severity and evidence. Four signal families are covered:

    Signal type   Source                                              Rule basis
    -----------   -------------------------------------------------   -------------------------------
    FAILURE       TASK_HISTORY, DYNAMIC_TABLE_REFRESH_HISTORY,        State = FAILED or error present
                  COPY_HISTORY
    LATENCY       TASK_HISTORY, QUERY_HISTORY                         Duration above LATENCY_SLO_SEC
                                                                      or above learned baseline
    FRESHNESS     INFORMATION_SCHEMA.DYNAMIC_TABLES(),                Lag above FRESHNESS_SLO_MIN
                  table LAST_ALTERED, DATA_METRIC FRESHNESS
    VOLUME        COPY_HISTORY, row count snapshots,                  Rows below MIN_EXPECTED_ROWS or
                  DATA_METRIC ROW_COUNT                               flagged by ANOMALY_DETECTION

## When to Use
- Adding a new pipeline or object type to monitoring.
- Tuning thresholds that produce too many or too few signals.
- Introducing ML baselines for workloads with seasonal volume.
- Investigating why an expected signal did not fire.

## Instructions
1. Create the signal table.

    CREATE TABLE IF NOT EXISTS OBS_DB.MONITOR.SIGNAL_EVENTS (
        SIGNAL_ID        STRING DEFAULT UUID_STRING(),
        SIGNAL_KEY       STRING,             -- deterministic dedup key
        PIPELINE_ID      STRING,
        SIGNAL_TYPE      STRING,             -- FAILURE | LATENCY | FRESHNESS | VOLUME
        SEVERITY         STRING,             -- CRITICAL | HIGH | MEDIUM | LOW
        OBSERVED_VALUE   FLOAT,
        THRESHOLD_VALUE  FLOAT,
        EVIDENCE         VARIANT,
        EVENT_TIME       TIMESTAMP_LTZ,
        DETECTED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        INCIDENT_ID      STRING
    );

2. Detect Task failures and latency breaches.

    MERGE INTO OBS_DB.MONITOR.SIGNAL_EVENTS t
    USING (
        SELECT
            SHA2(r.PIPELINE_ID || h.QUERY_ID || 'TASK') AS SIGNAL_KEY,
            r.PIPELINE_ID,
            IFF(h.STATE = 'FAILED', 'FAILURE', 'LATENCY') AS SIGNAL_TYPE,
            DATEDIFF('second', h.QUERY_START_TIME, h.COMPLETED_TIME) AS OBSERVED_VALUE,
            r.LATENCY_SLO_SEC AS THRESHOLD_VALUE,
            OBJECT_CONSTRUCT('state', h.STATE, 'error_code', h.ERROR_CODE,
                             'error_message', h.ERROR_MESSAGE,
                             'graph_run_group_id', h.GRAPH_RUN_GROUP_ID) AS EVIDENCE,
            h.COMPLETED_TIME AS EVENT_TIME
        FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY(
                 SCHEDULED_TIME_RANGE_START => DATEADD('hour', -1, CURRENT_TIMESTAMP()))) h
        JOIN OBS_DB.MONITOR.PIPELINE_REGISTRY r
          ON r.OBJECT_TYPE = 'TASK'
         AND r.OBJECT_FQN = h.DATABASE_NAME || '.' || h.SCHEMA_NAME || '.' || h.NAME
        WHERE r.IS_ACTIVE
          AND (h.STATE = 'FAILED'
               OR DATEDIFF('second', h.QUERY_START_TIME, h.COMPLETED_TIME) > r.LATENCY_SLO_SEC)
    ) s
    ON t.SIGNAL_KEY = s.SIGNAL_KEY
    WHEN NOT MATCHED THEN INSERT
        (SIGNAL_KEY, PIPELINE_ID, SIGNAL_TYPE, SEVERITY, OBSERVED_VALUE, THRESHOLD_VALUE, EVIDENCE, EVENT_TIME)
    VALUES
        (s.SIGNAL_KEY, s.PIPELINE_ID, s.SIGNAL_TYPE,
         IFF(s.SIGNAL_TYPE = 'FAILURE', 'HIGH', 'MEDIUM'),
         s.OBSERVED_VALUE, s.THRESHOLD_VALUE, s.EVIDENCE, s.EVENT_TIME);

3. Detect Dynamic Table freshness and refresh health.

    SELECT
        r.PIPELINE_ID,
        d.NAME,
        d.TARGET_LAG_SEC,
        d.MAXIMUM_LAG_SEC,
        d.TIME_WITHIN_TARGET_LAG_RATIO,
        d.SCHEDULING_STATE:state::STRING AS SCHEDULING_STATE,
        DATEDIFF('minute', d.LATEST_DATA_TIMESTAMP, CURRENT_TIMESTAMP()) AS DATA_AGE_MIN
    FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES()) d
    JOIN OBS_DB.MONITOR.PIPELINE_REGISTRY r
      ON r.OBJECT_TYPE = 'DYNAMIC_TABLE'
     AND r.OBJECT_FQN = d.DATABASE_NAME || '.' || d.SCHEMA_NAME || '.' || d.NAME
    WHERE DATEDIFF('minute', d.LATEST_DATA_TIMESTAMP, CURRENT_TIMESTAMP()) > r.FRESHNESS_SLO_MIN
       OR d.SCHEDULING_STATE:state::STRING <> 'ACTIVE';

    -- Refresh failures
    SELECT NAME, STATE, STATE_MESSAGE, REFRESH_START_TIME, REFRESH_END_TIME
    FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(
             ERROR_ONLY => TRUE,
             DATA_TIMESTAMP_START => DATEADD('hour', -1, CURRENT_TIMESTAMP())));

4. Detect volume drops on Snowpipe and batch loads.

    SELECT r.PIPELINE_ID, c.TABLE_NAME, SUM(c.ROW_COUNT) AS ROWS_LOADED, r.MIN_EXPECTED_ROWS
    FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY c
    JOIN OBS_DB.MONITOR.PIPELINE_REGISTRY r
      ON r.OBJECT_FQN = c.TABLE_CATALOG_NAME || '.' || c.TABLE_SCHEMA_NAME || '.' || c.TABLE_NAME
    WHERE c.LAST_LOAD_TIME >= DATEADD('hour', -24, CURRENT_TIMESTAMP())
    GROUP BY 1, 2, 4
    HAVING SUM(c.ROW_COUNT) < r.MIN_EXPECTED_ROWS;

5. Add ML baselines for seasonal workloads.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_DAILY_VOLUME AS
    SELECT PIPELINE_ID, DATE_TRUNC('hour', EVENT_TIME)::TIMESTAMP_NTZ AS TS, SUM(ROW_COUNT) AS ROWS
    FROM OBS_DB.MONITOR.LOAD_VOLUME_SNAPSHOTS
    GROUP BY 1, 2;

    CREATE OR REPLACE SNOWFLAKE.ML.ANOMALY_DETECTION OBS_DB.MONITOR.VOLUME_MODEL(
        INPUT_DATA => TABLE(SELECT * FROM OBS_DB.MONITOR.V_DAILY_VOLUME
                            WHERE TS < DATEADD('day', -1, CURRENT_DATE())),
        SERIES_COLNAME => 'PIPELINE_ID',
        TIMESTAMP_COLNAME => 'TS',
        TARGET_COLNAME => 'ROWS',
        LABEL_COLNAME => '');

    CALL OBS_DB.MONITOR.VOLUME_MODEL!DETECT_ANOMALIES(
        INPUT_DATA => TABLE(SELECT * FROM OBS_DB.MONITOR.V_DAILY_VOLUME
                            WHERE TS >= DATEADD('day', -1, CURRENT_DATE())),
        SERIES_COLNAME => 'PIPELINE_ID',
        TIMESTAMP_COLNAME => 'TS',
        TARGET_COLNAME => 'ROWS',
        CONFIG_OBJECT => {'prediction_interval': 0.99});

6. Wrap steps 2 to 5 in SP_DETECT_SIGNALS so the root Task calls one procedure.
7. Retrain anomaly models weekly with a separate Task; retain at least 4 weeks of history before enabling ML signals.

## Coding Conventions
- Build SIGNAL_KEY from pipeline ID, source query or refresh ID, and signal type so reruns never duplicate signals.
- Use INFORMATION_SCHEMA table functions for 5-minute checks and ACCOUNT_USAGE for lookbacks beyond 7 days.
- Always store raw evidence (error code, message, run IDs) in the EVIDENCE VARIANT for downstream triage.
- Default severity: FAILURE on tier 1 is CRITICAL; FAILURE elsewhere is HIGH; LATENCY and FRESHNESS are MEDIUM; VOLUME follows tier.
- Prefer static SLO rules for contractual thresholds and ML baselines for seasonal or bursty workloads; never mix both on the same metric without a documented precedence.
- Bound every lookback window explicitly to control scan cost.