---
name: cost-usage-anomaly-detection
description: This skill defines how the Cost Optimization Agent detects credit consumption spikes and budget overrun risk by comparing daily warehouse metering against rolling baselines, SNOWFLAKE.ML.ANOMALY_DETECTION forecasts, Budgets, and resource monitors, writing each finding to the COST_SIGNALS table. Use this skill for any task involving credit spike alerts, unexpected Snowflake bill increases, daily spend baselines, anomaly detection on WAREHOUSE_METERING_HISTORY, budget early warnings, resource monitor setup, or notifying finance teams and budget holders about overruns.
---

# Cost Usage Anomaly Detection

## Overview
This skill implements the spike and budget-risk portion of the Sense stage. It builds a daily credit series per warehouse, flags deviations against a 28-day baseline and an ML forecast, and projects month-end spend against budget so budget holders receive warnings before overruns occur.

## When to Use
- Credits jumped and the team needs to know which warehouse or service drove it.
- Setting up daily or hourly spike alerts.
- Configuring Budgets or resource monitors for early warning.
- Forecasting month-end spend.

## Instructions
1. Build the daily credit series view.

        CREATE OR REPLACE VIEW COST_OPS_DB.FINOPS.V_DAILY_WAREHOUSE_CREDITS AS
        SELECT
            WAREHOUSE_NAME,
            DATE_TRUNC('DAY', START_TIME)::DATE        AS USAGE_DATE,
            SUM(CREDITS_USED_COMPUTE)                  AS COMPUTE_CREDITS,
            SUM(CREDITS_USED_CLOUD_SERVICES)           AS CLOUD_SERVICES_CREDITS,
            SUM(CREDITS_USED)                          AS TOTAL_CREDITS
        FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
        WHERE START_TIME >= DATEADD('DAY', -90, CURRENT_DATE())
          AND START_TIME <  DATEADD('HOUR', -3, CURRENT_TIMESTAMP())
        GROUP BY 1, 2;

2. Detect statistical spikes with a rolling baseline (z-score above 3 and at least 25 percent above mean).

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_DETECT_USAGE_SPIKES()
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS OWNER
        AS
        $$
        BEGIN
            MERGE INTO COST_OPS_DB.FINOPS.COST_SIGNALS t
            USING (
                WITH b AS (
                    SELECT WAREHOUSE_NAME, USAGE_DATE, TOTAL_CREDITS,
                           AVG(TOTAL_CREDITS)    OVER (PARTITION BY WAREHOUSE_NAME ORDER BY USAGE_DATE
                                                       ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS BASE_AVG,
                           STDDEV(TOTAL_CREDITS) OVER (PARTITION BY WAREHOUSE_NAME ORDER BY USAGE_DATE
                                                       ROWS BETWEEN 28 PRECEDING AND 1 PRECEDING) AS BASE_SD
                    FROM COST_OPS_DB.FINOPS.V_DAILY_WAREHOUSE_CREDITS
                )
                SELECT WAREHOUSE_NAME, USAGE_DATE, TOTAL_CREDITS, BASE_AVG,
                       (TOTAL_CREDITS - BASE_AVG) / NULLIF(BASE_SD, 0) AS Z_SCORE
                FROM b
                WHERE USAGE_DATE >= DATEADD('DAY', -2, CURRENT_DATE())
                  AND BASE_AVG > 0
                  AND TOTAL_CREDITS > BASE_AVG * 1.25
                  AND (TOTAL_CREDITS - BASE_AVG) / NULLIF(BASE_SD, 0) > 3
            ) s
            ON  t.SIGNAL_TYPE = 'SPIKE'
            AND t.ENTITY_NAME = s.WAREHOUSE_NAME
            AND t.DETAILS:usage_date::DATE = s.USAGE_DATE
            WHEN NOT MATCHED THEN INSERT
                (SIGNAL_TYPE, ENTITY_TYPE, ENTITY_NAME, OBSERVED_CREDITS, BASELINE_CREDITS, DETAILS)
            VALUES
                ('SPIKE', 'WAREHOUSE', s.WAREHOUSE_NAME, s.TOTAL_CREDITS, s.BASE_AVG,
                 OBJECT_CONSTRUCT('usage_date', s.USAGE_DATE, 'z_score', s.Z_SCORE));
            RETURN 'Spike scan complete';
        END;
        $$;

3. Add ML-based anomaly detection for seasonal workloads.

        CREATE OR REPLACE VIEW COST_OPS_DB.FINOPS.V_ACCOUNT_DAILY_CREDITS AS
        SELECT USAGE_DATE::TIMESTAMP_NTZ AS TS, SUM(TOTAL_CREDITS) AS CREDITS
        FROM COST_OPS_DB.FINOPS.V_DAILY_WAREHOUSE_CREDITS GROUP BY 1;

        CREATE OR REPLACE SNOWFLAKE.ML.ANOMALY_DETECTION COST_OPS_DB.FINOPS.CREDIT_ANOMALY_MODEL(
            INPUT_DATA        => TABLE(SELECT * FROM COST_OPS_DB.FINOPS.V_ACCOUNT_DAILY_CREDITS
                                       WHERE TS < DATEADD('DAY', -7, CURRENT_DATE())),
            TIMESTAMP_COLNAME => 'TS',
            TARGET_COLNAME    => 'CREDITS',
            LABEL_COLNAME     => ''
        );

        CALL COST_OPS_DB.FINOPS.CREDIT_ANOMALY_MODEL!DETECT_ANOMALIES(
            INPUT_DATA        => TABLE(SELECT * FROM COST_OPS_DB.FINOPS.V_ACCOUNT_DAILY_CREDITS
                                       WHERE TS >= DATEADD('DAY', -7, CURRENT_DATE())),
            TIMESTAMP_COLNAME => 'TS',
            TARGET_COLNAME    => 'CREDITS'
        );

4. Project month-end spend and raise BUDGET_RISK signals when the run rate exceeds budget.

        SELECT
            SUM(TOTAL_CREDITS)                                         AS MTD_CREDITS,
            SUM(TOTAL_CREDITS) / DAY(CURRENT_DATE() - 1)
                * DAY(LAST_DAY(CURRENT_DATE()))                         AS PROJECTED_MONTH_CREDITS
        FROM COST_OPS_DB.FINOPS.V_DAILY_WAREHOUSE_CREDITS
        WHERE USAGE_DATE >= DATE_TRUNC('MONTH', CURRENT_DATE());

5. Back the forecast with hard controls: a resource monitor per critical warehouse and a Snowflake Budget for the account or custom group.

        CREATE OR REPLACE RESOURCE MONITOR RM_ANALYTICS_WH
            WITH CREDIT_QUOTA = 500 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
            TRIGGERS ON 75 PERCENT DO NOTIFY
                     ON 90 PERCENT DO NOTIFY
                     ON 100 PERCENT DO SUSPEND;
        ALTER WAREHOUSE ANALYTICS_WH SET RESOURCE_MONITOR = RM_ANALYTICS_WH;

6. Notify budget holders through a notification integration when new SPIKE or BUDGET_RISK signals appear.

        CALL SYSTEM$SEND_EMAIL(
            'FINOPS_EMAIL_INT',
            'finops-team@example.com',
            'Snowflake credit spike detected',
            'Review COST_OPS_DB.FINOPS.COST_SIGNALS for new SPIKE entries.'
        );

## Coding Conventions
- Exclude the latest 3 hours of ACCOUNT_USAGE data from every calculation.
- Keep thresholds (z-score, percentage uplift, baseline window) in a COST_CONFIG table, not hard-coded, for production deployments.
- Retrain the anomaly model weekly; never train on the window being scored.
- Resource monitors only suspend; recommend SUSPEND triggers but route their creation through the hand-off approval flow.