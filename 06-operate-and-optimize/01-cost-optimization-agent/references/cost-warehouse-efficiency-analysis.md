---
name: cost-warehouse-efficiency-analysis
description: This skill defines how the Cost Optimization Agent identifies idle, oversized, undersized, and misconfigured virtual warehouses by analyzing WAREHOUSE_METERING_HISTORY, WAREHOUSE_LOAD_HISTORY, WAREHOUSE_EVENTS_HISTORY, and SHOW WAREHOUSES output, producing IDLE_WAREHOUSE signals with current configuration context. Use this skill for any task involving idle warehouse detection, auto-suspend tuning, warehouse right-sizing, multi-cluster min and max settings, scaling policy review, queuing analysis, or reducing credits burned by warehouses running without queries.
---

# Cost Warehouse Efficiency Analysis

## Overview
Warehouses bill per second while running, with a 60-second minimum per resume. This skill finds credits consumed with little or no query activity, long auto-suspend windows, warehouses sized larger than their load requires, and multi-cluster settings that keep extra clusters alive.

## When to Use
- Detecting warehouses that run but execute few or no queries.
- Recommending AUTO_SUSPEND values.
- Evaluating whether a warehouse should be resized down or up.
- Reviewing multi-cluster MIN_CLUSTER_COUNT and SCALING_POLICY.

## Instructions
1. Capture current warehouse configuration into a snapshot table.

        SHOW WAREHOUSES;
        CREATE OR REPLACE TABLE COST_OPS_DB.FINOPS.WAREHOUSE_CONFIG_SNAPSHOT AS
        SELECT "name" AS WAREHOUSE_NAME, "size" AS WAREHOUSE_SIZE,
               "auto_suspend" AS AUTO_SUSPEND_SEC, "auto_resume" AS AUTO_RESUME,
               "min_cluster_count" AS MIN_CLUSTERS, "max_cluster_count" AS MAX_CLUSTERS,
               "scaling_policy" AS SCALING_POLICY, "resource_monitor" AS RESOURCE_MONITOR,
               CURRENT_TIMESTAMP() AS SNAPSHOT_AT
        FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()));

2. Compute idle credit share per warehouse over the last 14 days: hours with metered credits but negligible load.

        CREATE OR REPLACE VIEW COST_OPS_DB.FINOPS.V_WAREHOUSE_IDLE_ANALYSIS AS
        WITH m AS (
            SELECT WAREHOUSE_NAME, DATE_TRUNC('HOUR', START_TIME) AS HR, SUM(CREDITS_USED_COMPUTE) AS CREDITS
            FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
            WHERE START_TIME >= DATEADD('DAY', -14, CURRENT_TIMESTAMP())
              AND START_TIME <  DATEADD('HOUR', -3, CURRENT_TIMESTAMP())
            GROUP BY 1, 2
        ),
        l AS (
            SELECT WAREHOUSE_NAME, DATE_TRUNC('HOUR', START_TIME) AS HR,
                   AVG(AVG_RUNNING) AS AVG_RUNNING, AVG(AVG_QUEUED_LOAD) AS AVG_QUEUED
            FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_LOAD_HISTORY
            WHERE START_TIME >= DATEADD('DAY', -14, CURRENT_TIMESTAMP())
            GROUP BY 1, 2
        )
        SELECT m.WAREHOUSE_NAME,
               SUM(m.CREDITS)                                                    AS TOTAL_CREDITS,
               SUM(IFF(COALESCE(l.AVG_RUNNING, 0) < 0.05, m.CREDITS, 0))         AS IDLE_CREDITS,
               DIV0(SUM(IFF(COALESCE(l.AVG_RUNNING, 0) < 0.05, m.CREDITS, 0)),
                    SUM(m.CREDITS))                                              AS IDLE_RATIO,
               AVG(l.AVG_RUNNING)                                                AS AVG_RUNNING,
               MAX(l.AVG_QUEUED)                                                 AS PEAK_QUEUED
        FROM m LEFT JOIN l USING (WAREHOUSE_NAME, HR)
        GROUP BY 1;

3. Write IDLE_WAREHOUSE signals for warehouses above 30 percent idle ratio or with AUTO_SUSPEND above 300 seconds.

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_DETECT_IDLE_WAREHOUSES()
        RETURNS STRING LANGUAGE SQL EXECUTE AS OWNER
        AS
        $$
        BEGIN
            INSERT INTO COST_OPS_DB.FINOPS.COST_SIGNALS
                (SIGNAL_TYPE, ENTITY_TYPE, ENTITY_NAME, OBSERVED_CREDITS, BASELINE_CREDITS, DETAILS)
            SELECT 'IDLE_WAREHOUSE', 'WAREHOUSE', a.WAREHOUSE_NAME, a.IDLE_CREDITS, a.TOTAL_CREDITS,
                   OBJECT_CONSTRUCT('idle_ratio', a.IDLE_RATIO, 'avg_running', a.AVG_RUNNING,
                                    'peak_queued', a.PEAK_QUEUED, 'size', c.WAREHOUSE_SIZE,
                                    'auto_suspend_sec', c.AUTO_SUSPEND_SEC,
                                    'min_clusters', c.MIN_CLUSTERS, 'max_clusters', c.MAX_CLUSTERS,
                                    'window_days', 14)
            FROM COST_OPS_DB.FINOPS.V_WAREHOUSE_IDLE_ANALYSIS a
            JOIN COST_OPS_DB.FINOPS.WAREHOUSE_CONFIG_SNAPSHOT c USING (WAREHOUSE_NAME)
            WHERE a.TOTAL_CREDITS > 1
              AND (a.IDLE_RATIO > 0.30 OR COALESCE(c.AUTO_SUSPEND_SEC, 0) > 300
                   OR c.AUTO_SUSPEND_SEC IS NULL OR c.MIN_CLUSTERS > 1);
            RETURN 'Idle warehouse scan complete';
        END;
        $$;

4. Apply these sizing heuristics when drafting signals for modeling:
    - Downsize candidate: AVG_RUNNING low, no queuing, low spilling on its queries.
    - Upsize candidate: sustained queuing or heavy remote spilling; larger size can finish faster at similar total cost.
    - Multi-cluster: prefer MIN_CLUSTER_COUNT = 1 and SCALING_POLICY = 'ECONOMY' for non-interactive workloads.
    - Auto-suspend: 60 seconds for ad hoc and BI warehouses; longer only where local cache reuse is proven valuable.

5. Review resume frequency with WAREHOUSE_EVENTS_HISTORY before recommending very short suspend values, since frequent resumes each incur the 60-second minimum.

        SELECT WAREHOUSE_NAME, COUNT(*) AS RESUMES_14D
        FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_EVENTS_HISTORY
        WHERE EVENT_NAME = 'RESUME_WAREHOUSE'
          AND TIMESTAMP >= DATEADD('DAY', -14, CURRENT_TIMESTAMP())
        GROUP BY 1 ORDER BY 2 DESC;

## Coding Conventions
- Always pair metering data with load data; credits alone never prove idleness.
- Treat AUTO_SUSPEND NULL or 0 as never suspends and flag it as high priority.
- Refresh the configuration snapshot at the start of every scan.
- Never emit ALTER WAREHOUSE statements here; detection only.