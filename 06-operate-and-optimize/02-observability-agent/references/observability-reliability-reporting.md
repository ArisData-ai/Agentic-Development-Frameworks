---
name: observability-reliability-reporting
description: This skill defines how the Observability Agent turns incident, signal and escalation history into reliability metrics such as SLO attainment, mean time to detect, mean time to acknowledge, incident volume by priority and alert noise reduction, published as Snowflake views for operations, data product owners and leadership. Use this skill for any task involving reliability dashboards, platform health trends, SLO or SLA reporting, MTTD or MTTA calculation, noise reduction metrics, data product health scorecards, Streamlit or Snowsight dashboards, or weekly and monthly reliability reviews.
---
# Observability Reliability Reporting

## Overview
This skill turns operational history into trends for three audiences:

    Audience              Primary questions                                     Core views
    -------------------   ---------------------------------------------------   -------------------------------
    Operations teams      What is open, how fast are we detecting and           V_OPEN_INCIDENTS, V_MTTD_MTTA
                          acknowledging?
    Data product owners   Is my product fresh and meeting its SLO?              V_DATA_PRODUCT_HEALTH
    Leadership            Is reliability improving, and is noise going down?    V_RELIABILITY_TREND, V_NOISE_REDUCTION

## When to Use
- Building dashboards in Snowsight, Streamlit in Snowflake, Sigma or Power BI.
- Preparing weekly operations reviews or monthly leadership reliability reports.
- Measuring the value of the agent through noise reduction and detection speed.
- Tuning thresholds and impact weights based on override and false positive rates.

## Instructions
1. Measure detection and acknowledgment speed.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_MTTD_MTTA AS
    SELECT
        DATE_TRUNC('week', i.CREATED_AT) AS WEEK,
        i.PRIORITY,
        COUNT(*) AS INCIDENTS,
        AVG(DATEDIFF('minute', i.FIRST_SIGNAL_AT, i.CREATED_AT)) AS MTTD_MIN,
        AVG(DATEDIFF('minute', e.SENT_AT, e.ACKNOWLEDGED_AT)) AS MTTA_MIN
    FROM OBS_DB.MONITOR.INCIDENTS i
    LEFT JOIN OBS_DB.MONITOR.ESCALATIONS e ON e.INCIDENT_ID = i.INCIDENT_ID
    GROUP BY 1, 2;

2. Measure alert noise reduction.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_NOISE_REDUCTION AS
    SELECT
        DATE_TRUNC('day', s.DETECTED_AT) AS DAY,
        COUNT(*) AS RAW_SIGNALS,
        COUNT(DISTINCT s.INCIDENT_ID) AS INCIDENTS,
        ROUND(1 - COUNT(DISTINCT s.INCIDENT_ID) / NULLIF(COUNT(*), 0), 3) AS NOISE_REDUCTION_RATIO
    FROM OBS_DB.MONITOR.SIGNAL_EVENTS s
    GROUP BY 1;

3. Report data product health and SLO attainment.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_DATA_PRODUCT_HEALTH AS
    SELECT
        r.DATA_PRODUCT,
        r.OWNER_EMAIL,
        COUNT(DISTINCT IFF(s.SIGNAL_TYPE = 'FRESHNESS', s.SIGNAL_ID, NULL)) AS FRESHNESS_BREACHES_30D,
        COUNT(DISTINCT IFF(s.SIGNAL_TYPE = 'FAILURE', s.SIGNAL_ID, NULL)) AS FAILURES_30D,
        COUNT(DISTINCT IFF(s.SIGNAL_TYPE = 'VOLUME', s.SIGNAL_ID, NULL)) AS VOLUME_ANOMALIES_30D,
        COUNT(DISTINCT s.INCIDENT_ID) AS INCIDENTS_30D
    FROM OBS_DB.MONITOR.PIPELINE_REGISTRY r
    LEFT JOIN OBS_DB.MONITOR.SIGNAL_EVENTS s
      ON s.PIPELINE_ID = r.PIPELINE_ID
     AND s.DETECTED_AT >= DATEADD('day', -30, CURRENT_TIMESTAMP())
    GROUP BY 1, 2;

    -- Dynamic Table SLO attainment from refresh telemetry
    SELECT NAME, TARGET_LAG_SEC, MEAN_LAG_SEC, TIME_WITHIN_TARGET_LAG_RATIO
    FROM TABLE(INFORMATION_SCHEMA.DYNAMIC_TABLES())
    ORDER BY TIME_WITHIN_TARGET_LAG_RATIO ASC;

4. Report leadership trends.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_RELIABILITY_TREND AS
    SELECT
        DATE_TRUNC('month', CREATED_AT) AS MONTH,
        COUNT_IF(PRIORITY = 'P1') AS P1_INCIDENTS,
        COUNT_IF(PRIORITY = 'P2') AS P2_INCIDENTS,
        COUNT(*) AS TOTAL_INCIDENTS,
        AVG(DATEDIFF('minute', FIRST_SIGNAL_AT, CREATED_AT)) AS MTTD_MIN
    FROM OBS_DB.MONITOR.INCIDENTS
    GROUP BY 1;

5. Track priority override rate to tune impact weights.

    SELECT AGENT_PRIORITY, CONFIRMED_PRIORITY, COUNT(*) AS N
    FROM OBS_DB.MONITOR.ESCALATIONS
    WHERE CONFIRMED_PRIORITY IS NOT NULL
    GROUP BY 1, 2
    ORDER BY N DESC;

6. Publish views to a REPORTING role and build dashboards on top; generate a weekly narrative summary with SNOWFLAKE.CORTEX.COMPLETE over V_RELIABILITY_TREND and V_NOISE_REDUCTION for leadership email.
7. Apply data retention: keep SIGNAL_EVENTS for 90 days and INCIDENTS and ESCALATIONS for 13 months to support year-over-year trends.

## Coding Conventions
- Reporting views are read-only and prefixed V_; grant SELECT to OBS_REPORTING_ROLE only.
- Define metric formulas once in views; dashboards must not reimplement calculations.
- Use calendar boundaries (week, month) in UTC unless the organization standard differs.
- Treat priority override rate above 20 percent as a signal to retune impact weights in the correlation config.
- Exclude suppressed and test pipelines from leadership metrics through IS_ACTIVE and a pipeline environment flag.