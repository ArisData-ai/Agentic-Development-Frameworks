---
name: remediation-incident-intake
description: This skill defines incident intake for the Remediation Agent, which detects failed tasks, stalled Snowpipe loads, stale streams, lagging dynamic tables, and breached SLA thresholds and writes normalized records to an INCIDENTS table through Snowflake Alerts, TASK_HISTORY, and ACCOUNT_USAGE views. Use this skill for any task involving incident detection, failed task monitoring, pipeline stall checks, stream staleness checks, freshness or SLA breach alerts, deduplicating incidents, or feeding triaged incidents into an automated remediation workflow.
---
# Remediation Incident Intake

## Overview
Intake converts raw operational signals into deduplicated, severity-tagged incident rows. It is the agent's only entry point; diagnosis and execution never act on signals that have not passed through intake.

## When to Use
- Adding a new incident source (task, pipe, stream, dynamic table, custom threshold)
- Tuning detection frequency or deduplication windows
- Mapping severity from SLA tiers of data products

## Instructions
1. Detect failed tasks from the INFORMATION_SCHEMA table function for low latency.

        CREATE OR REPLACE ALERT OPS_DB.REMEDIATION.ALERT_FAILED_TASKS
            WAREHOUSE = REMEDIATION_WH
            SCHEDULE = '5 MINUTE'
            IF (EXISTS (
                SELECT 1
                FROM TABLE(PROD_DB.INFORMATION_SCHEMA.TASK_HISTORY(
                    SCHEDULED_TIME_RANGE_START => DATEADD('minute', -10, CURRENT_TIMESTAMP())))
                WHERE state = 'FAILED'))
            THEN
                INSERT INTO OPS_DB.REMEDIATION.INCIDENTS
                    (source_type, object_name, severity, error_code, error_message, context)
                SELECT 'TASK',
                       database_name || '.' || schema_name || '.' || name,
                       'SEV2', error_code, error_message,
                       OBJECT_CONSTRUCT('query_id', query_id, 'run_id', run_id,
                                        'scheduled_time', scheduled_time)
                FROM TABLE(PROD_DB.INFORMATION_SCHEMA.TASK_HISTORY(
                    SCHEDULED_TIME_RANGE_START => DATEADD('minute', -10, CURRENT_TIMESTAMP())))
                WHERE state = 'FAILED'
                  AND NOT EXISTS (
                      SELECT 1 FROM OPS_DB.REMEDIATION.INCIDENTS i
                      WHERE i.context:query_id::STRING = query_id);

        ALTER ALERT OPS_DB.REMEDIATION.ALERT_FAILED_TASKS RESUME;

2. Detect stalled pipelines by comparing last load time against expected cadence.

        SELECT pipe_name, MAX(last_load_time) AS last_load
        FROM SNOWFLAKE.ACCOUNT_USAGE.COPY_HISTORY
        WHERE last_load_time > DATEADD('day', -1, CURRENT_TIMESTAMP())
        GROUP BY pipe_name
        HAVING DATEDIFF('minute', MAX(last_load_time), CURRENT_TIMESTAMP()) > 60;

3. Detect stale streams and lagging dynamic tables.

        SHOW STREAMS IN DATABASE PROD_DB;
        SELECT "name", "stale", "stale_after"
        FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
        WHERE "stale" = 'true'
           OR "stale_after" < DATEADD('day', 1, CURRENT_TIMESTAMP());

        SELECT name, state, state_message, data_timestamp
        FROM TABLE(PROD_DB.INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY())
        WHERE state = 'FAILED';

4. Register custom SLA thresholds (row counts, freshness, error rates) in a config table and evaluate them in one Alert.
5. Deduplicate on object_name plus error_code within a rolling window; increment an occurrence count instead of inserting duplicates.
6. Assign severity from the data product SLA tier, not from the error alone.

## Coding Conventions
- Use INFORMATION_SCHEMA table functions for detection under 15 minutes; use ACCOUNT_USAGE only for history and reporting given its latency.
- Name alerts ALERT_<SIGNAL>; name config tables <SIGNAL>_THRESHOLDS.
- Always store the originating query_id or run_id in context for traceability.
- Keep intake read-only against production objects.