---
name: observability-agent
description: This skill defines the end-to-end Observability Agent for Snowflake, which ingests pipeline execution events from Streams, Tasks, Dynamic Tables and query history, detects latency, failure, freshness and volume anomalies, correlates them into prioritized incidents with Cortex Agents, and escalates a triaged view to the Operations Lead. Use this skill for any task involving pipeline monitoring, alert noise reduction, incident triage, mean time to detect improvements, observability schema setup, Snowflake Alerts, or building the Operate and Optimize monitoring layer for alert-heavy data platforms.
---
# Observability Agent

## Overview
The Observability Agent monitors the health and performance of Snowflake pipelines and workloads so problems surface before consumers notice them. It runs a three-stage loop:

- Sense: detect latency, failure, freshness and volume anomalies across registered pipelines.
- Act: correlate related signals, group duplicate alerts, and rank incidents by business impact.
- Hand off: escalate the triaged incident view to the Operations Lead, who confirms priority and assigns response.

The agent replaces raw alert streams with a small set of prioritized incidents. Operations teams work from incidents, data product owners learn of issues early, and leadership sees reliability trends.

Reference architecture:

    Source telemetry                    Agent layer                         Consumers
    -------------------------------     -------------------------------     ----------------------
    ACCOUNT_USAGE.TASK_HISTORY    -->   OBS_DB.MONITOR.SIGNAL_EVENTS  -->   Operations Lead
    DYNAMIC_TABLE_REFRESH_HISTORY -->   OBS_DB.MONITOR.INCIDENTS      -->   Data product owners
    QUERY_HISTORY / COPY_HISTORY  -->   Cortex Agent triage           -->   Leadership dashboards
    Event table (logs, traces)    -->   Snowflake Alerts + notify

## When to Use
- Standing up monitoring for an account with many Tasks, Dynamic Tables, Snowpipes or dbt jobs.
- Teams report alert fatigue or cannot separate critical incidents from noise.
- Data product owners need early warning on stale or incomplete data.
- Leadership needs reliability KPIs such as MTTD, incident counts and SLO attainment.
- Downstream of the Pipeline, CI/CD and Deployment Validation agents; upstream of the Remediation Agent.

## Instructions
1. Provision the observability foundation.

    CREATE DATABASE IF NOT EXISTS OBS_DB;
    CREATE SCHEMA IF NOT EXISTS OBS_DB.MONITOR;
    CREATE WAREHOUSE IF NOT EXISTS OBS_WH
        WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;
    CREATE ROLE IF NOT EXISTS OBS_AGENT_ROLE;
    GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE OBS_AGENT_ROLE;
    GRANT USAGE, OPERATE ON WAREHOUSE OBS_WH TO ROLE OBS_AGENT_ROLE;
    GRANT ALL ON SCHEMA OBS_DB.MONITOR TO ROLE OBS_AGENT_ROLE;
    GRANT MONITOR EXECUTION ON ACCOUNT TO ROLE OBS_AGENT_ROLE;

2. Register every monitored pipeline with ownership, criticality and SLO thresholds.

    CREATE TABLE IF NOT EXISTS OBS_DB.MONITOR.PIPELINE_REGISTRY (
        PIPELINE_ID          STRING PRIMARY KEY,
        OBJECT_TYPE          STRING,          -- TASK | DYNAMIC_TABLE | PIPE | DBT_MODEL
        OBJECT_FQN           STRING,
        DATA_PRODUCT         STRING,
        OWNER_EMAIL          STRING,
        BUSINESS_TIER        NUMBER(1),       -- 1 = critical, 3 = low
        FRESHNESS_SLO_MIN    NUMBER,
        LATENCY_SLO_SEC      NUMBER,
        MIN_EXPECTED_ROWS    NUMBER,
        UPSTREAM_IDS         ARRAY,
        IS_ACTIVE            BOOLEAN DEFAULT TRUE
    );

3. Deploy signal detection as described in `observability-signal-detection`.
4. Deploy correlation and impact scoring as described in `observability-incident-correlation`.
5. Configure escalation, acknowledgment and assignment as described in `observability-escalation-handoff`.
6. Publish reliability trends as described in `observability-reliability-reporting`.
7. Orchestrate the loop with a single root Task graph so stages run in order.

    CREATE OR REPLACE TASK OBS_DB.MONITOR.T_OBS_ROOT
        WAREHOUSE = OBS_WH
        SCHEDULE = 'USING CRON */5 * * * * UTC'
    AS CALL OBS_DB.MONITOR.SP_DETECT_SIGNALS();

    CREATE OR REPLACE TASK OBS_DB.MONITOR.T_OBS_CORRELATE
        WAREHOUSE = OBS_WH AFTER OBS_DB.MONITOR.T_OBS_ROOT
    AS CALL OBS_DB.MONITOR.SP_CORRELATE_INCIDENTS();

    CREATE OR REPLACE TASK OBS_DB.MONITOR.T_OBS_ESCALATE
        WAREHOUSE = OBS_WH AFTER OBS_DB.MONITOR.T_OBS_CORRELATE
    AS CALL OBS_DB.MONITOR.SP_ESCALATE_INCIDENTS();

    ALTER TASK OBS_DB.MONITOR.T_OBS_ESCALATE RESUME;
    ALTER TASK OBS_DB.MONITOR.T_OBS_CORRELATE RESUME;
    ALTER TASK OBS_DB.MONITOR.T_OBS_ROOT RESUME;

8. Validate on a non-production account by forcing one failed Task, one stale Dynamic Table and one empty load, then confirm that exactly one incident per root cause reaches the Operations Lead.

## Coding Conventions
- Keep all agent objects in OBS_DB.MONITOR; never write to monitored application schemas.
- Use uppercase SQL keywords and object names; use snake-free, uppercase column names.
- Prefix procedures with SP_, Tasks with T_, Alerts with ALERT_, views with V_.
- Store all thresholds in PIPELINE_REGISTRY; never hardcode thresholds in procedures.
- Read ACCOUNT_USAGE for history and trends (up to 45 minutes latency); use INFORMATION_SCHEMA table functions for near real-time checks.
- Make every procedure idempotent with MERGE on deterministic keys.
- The agent observes, correlates and escalates; it never restarts, suspends or modifies monitored pipelines. Corrective action belongs to the Remediation Agent.
- Run the agent on a dedicated XSMALL warehouse with AUTO_SUSPEND = 60 to cap monitoring cost.

## References
- `references/observability-signal-detection.md`: Collecting telemetry from Tasks, Dynamic Tables, Snowpipe and query history; latency, failure, freshness and volume rules; ML anomaly detection baselines; the SIGNAL_EVENTS schema.
- `references/observability-incident-correlation.md`: Deduplication, lineage-based grouping, root-cause selection, business impact scoring, and Cortex Agent triage summaries; the INCIDENTS schema.
- `references/observability-escalation-handoff.md`: Snowflake Alerts, notification integrations, escalation payloads, Operations Lead acknowledgment and assignment, and suppression windows.
- `references/observability-reliability-reporting.md`: SLO attainment, MTTD and MTTA metrics, noise reduction ratios, and leadership and data product owner dashboards.