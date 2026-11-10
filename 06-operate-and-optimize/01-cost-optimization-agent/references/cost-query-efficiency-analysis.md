---
name: cost-query-efficiency-analysis
description: This skill defines how the Cost Optimization Agent finds costly and inefficient query patterns by aggregating QUERY_HISTORY and QUERY_ATTRIBUTION_HISTORY by parameterized hash, user, role, and warehouse, detecting spilling, poor partition pruning, queuing, and repeated full scans, then writing INEFFICIENT_QUERY signals. Use this skill for any task involving expensive query identification, per-user or per-team cost attribution, chargeback and showback reports, spill and pruning diagnostics, clustering or materialized view candidates, or explaining who drives Snowflake credit consumption.
---

# Cost Query Efficiency Analysis

## Overview
This skill attributes compute credits to query patterns and owners, then flags patterns whose execution profile indicates waste. It answers the core FinOps question of who drives spend and why.

## When to Use
- Ranking the most expensive recurring queries.
- Building showback or chargeback by user, role, or query tag.
- Diagnosing spilling, pruning, or queuing problems.
- Feeding query rewrite or clustering recommendations to the modeling stage.

## Instructions
1. Attribute credits to recurring query patterns.

        CREATE OR REPLACE VIEW COST_OPS_DB.FINOPS.V_QUERY_PATTERN_COST AS
        SELECT
            q.QUERY_PARAMETERIZED_HASH,
            ANY_VALUE(LEFT(q.QUERY_TEXT, 500))                     AS SAMPLE_TEXT,
            q.WAREHOUSE_NAME,
            q.WAREHOUSE_SIZE,
            ANY_VALUE(q.USER_NAME)                                  AS SAMPLE_USER,
            ANY_VALUE(q.ROLE_NAME)                                  AS SAMPLE_ROLE,
            ANY_VALUE(q.QUERY_TAG)                                  AS QUERY_TAG,
            COUNT(*)                                                AS EXECUTIONS,
            SUM(a.CREDITS_ATTRIBUTED_COMPUTE)                       AS ATTRIBUTED_CREDITS,
            AVG(q.EXECUTION_TIME) / 1000                            AS AVG_EXEC_SEC,
            SUM(q.BYTES_SPILLED_TO_LOCAL_STORAGE)                   AS LOCAL_SPILL_BYTES,
            SUM(q.BYTES_SPILLED_TO_REMOTE_STORAGE)                  AS REMOTE_SPILL_BYTES,
            DIV0(SUM(q.PARTITIONS_SCANNED), SUM(q.PARTITIONS_TOTAL)) AS SCAN_RATIO,
            AVG(q.QUEUED_OVERLOAD_TIME) / 1000                      AS AVG_QUEUED_SEC
        FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q
        JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY a
          ON a.QUERY_ID = q.QUERY_ID
        WHERE q.START_TIME >= DATEADD('DAY', -14, CURRENT_TIMESTAMP())
          AND q.START_TIME <  DATEADD('HOUR', -3, CURRENT_TIMESTAMP())
          AND q.EXECUTION_STATUS = 'SUCCESS'
        GROUP BY q.QUERY_PARAMETERIZED_HASH, q.WAREHOUSE_NAME, q.WAREHOUSE_SIZE;

2. Classify inefficiency with explicit rules and persist signals.

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_DETECT_INEFFICIENT_QUERIES()
        RETURNS STRING LANGUAGE SQL EXECUTE AS OWNER
        AS
        $$
        BEGIN
            INSERT INTO COST_OPS_DB.FINOPS.COST_SIGNALS
                (SIGNAL_TYPE, ENTITY_TYPE, ENTITY_NAME, OBSERVED_CREDITS, DETAILS)
            SELECT 'INEFFICIENT_QUERY', 'QUERY_PATTERN', QUERY_PARAMETERIZED_HASH, ATTRIBUTED_CREDITS,
                   OBJECT_CONSTRUCT(
                       'warehouse', WAREHOUSE_NAME, 'size', WAREHOUSE_SIZE,
                       'executions', EXECUTIONS, 'avg_exec_sec', AVG_EXEC_SEC,
                       'sample_user', SAMPLE_USER, 'sample_role', SAMPLE_ROLE,
                       'query_tag', QUERY_TAG, 'sample_text', SAMPLE_TEXT,
                       'issues', ARRAY_CONSTRUCT_COMPACT(
                           IFF(REMOTE_SPILL_BYTES > 0, 'REMOTE_SPILL', NULL),
                           IFF(LOCAL_SPILL_BYTES > 10 * POWER(1024, 3), 'LOCAL_SPILL', NULL),
                           IFF(SCAN_RATIO > 0.8 AND EXECUTIONS > 20, 'POOR_PRUNING', NULL),
                           IFF(AVG_QUEUED_SEC > 30, 'QUEUING', NULL),
                           IFF(EXECUTIONS > 500, 'HIGH_FREQUENCY', NULL)))
            FROM COST_OPS_DB.FINOPS.V_QUERY_PATTERN_COST
            WHERE ATTRIBUTED_CREDITS > 0.5
            QUALIFY ROW_NUMBER() OVER (ORDER BY ATTRIBUTED_CREDITS DESC) <= 100;
            RETURN 'Query scan complete';
        END;
        $$;

3. Map each issue to a recommendation type for the modeling stage:
    - REMOTE_SPILL or LOCAL_SPILL: upsize the warehouse for that workload or reduce intermediate data volume.
    - POOR_PRUNING: add filters on clustered columns, define a clustering key, or enable search optimization for point lookups.
    - QUEUING: enable multi-cluster or isolate the workload onto its own warehouse.
    - HIGH_FREQUENCY: evaluate result cache reuse, materialized views, dynamic tables, or reduced refresh frequency.

4. Produce showback by owner for finance teams.

        SELECT a.USER_NAME, q.ROLE_NAME, COALESCE(NULLIF(q.QUERY_TAG, ''), 'UNTAGGED') AS QUERY_TAG,
               SUM(a.CREDITS_ATTRIBUTED_COMPUTE) AS CREDITS_30D
        FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY a
        JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q ON q.QUERY_ID = a.QUERY_ID
        WHERE a.START_TIME >= DATEADD('DAY', -30, CURRENT_TIMESTAMP())
        GROUP BY 1, 2, 3 ORDER BY CREDITS_30D DESC;

## Coding Conventions
- Aggregate by QUERY_PARAMETERIZED_HASH so literal-only variations collapse into one pattern.
- Truncate QUERY_TEXT before storing; never persist full query text containing literals beyond 500 characters.
- QUERY_ATTRIBUTION_HISTORY excludes warehouse idle time; combine with idle analysis for full warehouse cost.
- Encourage QUERY_TAG standards (team, application, pipeline) so attribution maps to business owners.