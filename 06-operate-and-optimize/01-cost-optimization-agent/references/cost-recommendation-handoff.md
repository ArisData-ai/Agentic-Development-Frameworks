---
name: cost-recommendation-handoff
description: This skill defines how the Cost Optimization Agent presents ranked COST_RECOMMENDATIONS to the Operations Lead, captures approve, adjust, or decline decisions in an audited COST_ACTION_LOG, applies only approved warehouse changes through a controlled stored procedure, and supports rollback. Use this skill for any task involving human-in-the-loop approval, ranked savings action lists, Operations Lead review queues, applying approved ALTER WAREHOUSE changes, decision audit trails, rollback of cost changes, or weekly FinOps savings reports.
---

# Cost Recommendation Handoff

## Overview
The agent recommends; the Operations Lead decides. This skill implements the Hand-off stage with a ranked review queue, a single decision procedure that records every choice, controlled application of approved SQL, and rollback from stored configuration.

## When to Use
- Building the review queue or weekly savings digest.
- Recording approvals, adjustments, or declines.
- Applying or reverting an approved change.
- Reporting realized versus estimated savings.

## Instructions
1. Expose a ranked review queue.

        CREATE OR REPLACE VIEW COST_OPS_DB.FINOPS.V_REVIEW_QUEUE AS
        SELECT REC_ID, ACTION_TYPE, TARGET_OBJECT, CURRENT_CONFIG, PROPOSED_CONFIG, PROPOSED_SQL,
               EST_MONTHLY_SAVINGS_CR, PERFORMANCE_RISK, CONFIDENCE, PRIORITY_SCORE,
               RANK() OVER (ORDER BY PRIORITY_SCORE DESC) AS PRIORITY_RANK, CREATED_AT
        FROM COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS
        WHERE STATUS = 'PENDING';

        GRANT SELECT ON VIEW COST_OPS_DB.FINOPS.V_REVIEW_QUEUE TO ROLE FINOPS_APPROVER_ROLE;

2. Record decisions and apply approved changes through one procedure owned by a role with MODIFY on target warehouses, granted only to FINOPS_APPROVER_ROLE.

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_DECIDE_RECOMMENDATION(
            P_REC_ID STRING, P_DECISION STRING, P_ADJUSTED_SQL STRING, P_NOTES STRING)
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS OWNER
        AS
        $$
        DECLARE
            v_sql    STRING;
            v_status STRING;
        BEGIN
            SELECT STATUS, PROPOSED_SQL INTO :v_status, :v_sql
            FROM COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS WHERE REC_ID = :P_REC_ID;

            IF (v_status <> 'PENDING') THEN
                RETURN 'Recommendation is not pending';
            END IF;
            IF (P_DECISION NOT IN ('APPROVED', 'ADJUSTED', 'DECLINED')) THEN
                RETURN 'Invalid decision';
            END IF;
            IF (P_DECISION = 'ADJUSTED') THEN
                v_sql := P_ADJUSTED_SQL;
            END IF;
            IF (P_DECISION <> 'DECLINED' AND NOT REGEXP_LIKE(v_sql, '^ALTER WAREHOUSE [A-Z0-9_]+ SET .*', 'i')) THEN
                RETURN 'Only ALTER WAREHOUSE ... SET statements may be applied automatically';
            END IF;

            INSERT INTO COST_OPS_DB.FINOPS.COST_ACTION_LOG (REC_ID, DECISION, FINAL_SQL, NOTES)
            VALUES (:P_REC_ID, :P_DECISION, :v_sql, :P_NOTES);

            IF (P_DECISION = 'DECLINED') THEN
                UPDATE COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS SET STATUS = 'DECLINED' WHERE REC_ID = :P_REC_ID;
                RETURN 'Declined and logged';
            END IF;

            EXECUTE IMMEDIATE :v_sql;
            UPDATE COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS SET STATUS = 'APPLIED' WHERE REC_ID = :P_REC_ID;
            RETURN 'Applied: ' || v_sql;
        END;
        $$;

        GRANT USAGE ON PROCEDURE COST_OPS_DB.FINOPS.SP_DECIDE_RECOMMENDATION(STRING, STRING, STRING, STRING)
            TO ROLE FINOPS_APPROVER_ROLE;

3. Route QUERY_REWRITE recommendations (no PROPOSED_SQL) to the owning team as tickets; the Operations Lead marks them APPROVED to assign, and the agent never executes them.

4. Provide rollback from CURRENT_CONFIG.

        SELECT REC_ID, TARGET_OBJECT,
               CASE ACTION_TYPE
                   WHEN 'AUTO_SUSPEND'  THEN 'ALTER WAREHOUSE ' || TARGET_OBJECT || ' SET AUTO_SUSPEND = '
                                             || COALESCE(CURRENT_CONFIG:auto_suspend_sec::STRING, 'NULL')
                   WHEN 'RESIZE'        THEN 'ALTER WAREHOUSE ' || TARGET_OBJECT || ' SET WAREHOUSE_SIZE = '
                                             || CURRENT_CONFIG:size::STRING
                   WHEN 'MULTI_CLUSTER' THEN 'ALTER WAREHOUSE ' || TARGET_OBJECT || ' SET MIN_CLUSTER_COUNT = '
                                             || CURRENT_CONFIG:min_clusters::STRING
               END AS ROLLBACK_SQL
        FROM COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS
        WHERE STATUS = 'APPLIED';

5. Publish a weekly digest for FinOps and budget holders: top 10 pending actions, approved savings, and realized savings measured from WAREHOUSE_METERING_HISTORY 14 days before and after each change.

## Coding Conventions
- Present actions ranked by PRIORITY_SCORE with savings, risk, and confidence side by side.
- One decision per recommendation; decided recommendations are immutable, and re-proposals create new rows.
- Allow-list executable statement shapes; reject anything other than ALTER WAREHOUSE ... SET.
- Log the approver through CURRENT_USER() inside the procedure, never from a parameter.