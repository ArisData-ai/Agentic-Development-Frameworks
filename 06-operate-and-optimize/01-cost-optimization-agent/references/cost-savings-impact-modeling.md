---
name: cost-savings-impact-modeling
description: This skill defines how the Cost Optimization Agent converts COST_SIGNALS into quantified COST_RECOMMENDATIONS by running Snowpark Python procedures that estimate monthly credit savings, performance risk, and confidence for warehouse resizing, auto-suspend changes, multi-cluster adjustments, and query remediation. Use this skill for any task involving what-if cost modeling, savings estimates, warehouse resize simulation, auto-suspend impact analysis, performance trade-off scoring, generating proposed ALTER statements, or building the business case for a Snowflake optimization.
---

# Cost Savings Impact Modeling

## Overview
Detection identifies waste; modeling turns it into a decision. This skill implements the Act stage: for each open signal it computes current cost, proposed configuration, expected savings, performance risk, and the exact SQL that would apply the change, then writes a PENDING recommendation.

## When to Use
- Estimating savings before changing a warehouse.
- Comparing several candidate configurations for one warehouse.
- Producing the PROPOSED_SQL and risk rating for the hand-off stage.

## Instructions
1. Use the standard credit-per-hour table for size math.

        XSMALL 1 | SMALL 2 | MEDIUM 4 | LARGE 8 | XLARGE 16 | X2LARGE 32 | X3LARGE 64 | X4LARGE 128

2. Apply these modeling rules:
    - Auto-suspend: savings approximately equal idle credits multiplied by the share of idle time beyond the proposed suspend window; subtract added resume minimums from WAREHOUSE_EVENTS_HISTORY patterns.
    - Downsize one step: compute credits halve per running hour; assume execution time rises up to 2x for compute-bound queries, so savings are bounded between 0 and 50 percent; mark risk MEDIUM if any pattern spills or queues.
    - Upsize for spilling: credits per hour double, but runtime often falls by more than half when remote spill disappears; recommend only when modeled cost is neutral or better.
    - Multi-cluster: savings equal credits from clusters above the proposed minimum during low-load hours.
    - Query remediation: savings equal attributed credits multiplied by an expected reduction factor (pruning 0.5, spill 0.4, frequency 0.6); confidence stays at or below 0.6 until validated.

3. Implement the modeling procedure in Snowpark.

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_MODEL_AND_RANK_RECOMMENDATIONS()
        RETURNS STRING
        LANGUAGE PYTHON
        RUNTIME_VERSION = '3.11'
        PACKAGES = ('snowflake-snowpark-python')
        HANDLER = 'run'
        EXECUTE AS OWNER
        AS
        $$
        import json
        from snowflake.snowpark import Session

        SIZES = ["XSMALL", "SMALL", "MEDIUM", "LARGE", "XLARGE", "X2LARGE", "X3LARGE", "X4LARGE"]
        RISK_WEIGHT = {"LOW": 1.0, "MEDIUM": 0.6, "HIGH": 0.3}

        def norm_size(s):
            return (s or "").upper().replace("-", "").replace(" ", "")

        def model_idle(sig, d):
            recs = []
            days = d.get("window_days", 14)
            monthly_idle = float(sig["OBSERVED_CREDITS"]) * 30 / days
            suspend = d.get("auto_suspend_sec")
            if suspend is None or suspend == 0 or suspend > 60:
                recs.append(dict(
                    ACTION_TYPE="AUTO_SUSPEND",
                    CURRENT_CONFIG={"auto_suspend_sec": suspend},
                    PROPOSED_CONFIG={"auto_suspend_sec": 60},
                    PROPOSED_SQL=f"ALTER WAREHOUSE {sig['ENTITY_NAME']} SET AUTO_SUSPEND = 60",
                    EST=monthly_idle * 0.7, RISK="LOW", CONF=0.8))
            size = norm_size(d.get("size"))
            if size in SIZES and SIZES.index(size) > 0 and (d.get("peak_queued") or 0) == 0 \
                    and (d.get("avg_running") or 0) < 0.3:
                target = SIZES[SIZES.index(size) - 1]
                monthly_total = float(sig["BASELINE_CREDITS"]) * 30 / days
                recs.append(dict(
                    ACTION_TYPE="RESIZE",
                    CURRENT_CONFIG={"size": size},
                    PROPOSED_CONFIG={"size": target},
                    PROPOSED_SQL=f"ALTER WAREHOUSE {sig['ENTITY_NAME']} SET WAREHOUSE_SIZE = {target}",
                    EST=(monthly_total - monthly_idle) * 0.25, RISK="MEDIUM", CONF=0.6))
            if (d.get("min_clusters") or 1) > 1:
                recs.append(dict(
                    ACTION_TYPE="MULTI_CLUSTER",
                    CURRENT_CONFIG={"min_clusters": d.get("min_clusters")},
                    PROPOSED_CONFIG={"min_clusters": 1, "scaling_policy": "ECONOMY"},
                    PROPOSED_SQL=(f"ALTER WAREHOUSE {sig['ENTITY_NAME']} SET MIN_CLUSTER_COUNT = 1 "
                                  f"SCALING_POLICY = 'ECONOMY'"),
                    EST=monthly_idle * 0.5, RISK="MEDIUM", CONF=0.6))
            return recs

        def model_query(sig, d):
            factors = {"POOR_PRUNING": 0.5, "REMOTE_SPILL": 0.4, "LOCAL_SPILL": 0.2,
                       "HIGH_FREQUENCY": 0.6, "QUEUING": 0.1}
            issues = d.get("issues") or []
            if not issues:
                return []
            factor = max(factors.get(i, 0) for i in issues)
            monthly = float(sig["OBSERVED_CREDITS"]) * 30 / 14
            return [dict(
                ACTION_TYPE="QUERY_REWRITE",
                CURRENT_CONFIG={"warehouse": d.get("warehouse"), "issues": issues},
                PROPOSED_CONFIG={"remediation": issues},
                PROPOSED_SQL=None,
                EST=monthly * factor, RISK="LOW", CONF=0.5)]

        def run(session: Session) -> str:
            signals = session.sql("""
                SELECT s.* FROM COST_OPS_DB.FINOPS.COST_SIGNALS s
                LEFT JOIN COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS r ON r.SIGNAL_ID = s.SIGNAL_ID
                WHERE r.SIGNAL_ID IS NULL
                  AND s.SIGNAL_TYPE IN ('IDLE_WAREHOUSE', 'INEFFICIENT_QUERY')
                  AND s.DETECTED_AT >= DATEADD('DAY', -1, CURRENT_TIMESTAMP())
            """).collect()
            rows = []
            for sig in signals:
                d = json.loads(sig["DETAILS"]) if sig["DETAILS"] else {}
                recs = model_idle(sig, d) if sig["SIGNAL_TYPE"] == "IDLE_WAREHOUSE" else model_query(sig, d)
                for r in recs:
                    score = r["EST"] * r["CONF"] * RISK_WEIGHT[r["RISK"]]
                    rows.append((sig["SIGNAL_ID"], r["ACTION_TYPE"], sig["ENTITY_NAME"],
                                 json.dumps(r["CURRENT_CONFIG"]), json.dumps(r["PROPOSED_CONFIG"]),
                                 r["PROPOSED_SQL"], round(r["EST"], 4), r["RISK"], r["CONF"], round(score, 4)))
            if rows:
                df = session.create_dataframe(rows, schema=[
                    "SIGNAL_ID", "ACTION_TYPE", "TARGET_OBJECT", "CURRENT_CONFIG_STR", "PROPOSED_CONFIG_STR",
                    "PROPOSED_SQL", "EST_MONTHLY_SAVINGS_CR", "PERFORMANCE_RISK", "CONFIDENCE", "PRIORITY_SCORE"])
                df.create_or_replace_temp_view("NEW_RECS")
                session.sql("""
                    INSERT INTO COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS
                        (SIGNAL_ID, ACTION_TYPE, TARGET_OBJECT, CURRENT_CONFIG, PROPOSED_CONFIG,
                         PROPOSED_SQL, EST_MONTHLY_SAVINGS_CR, PERFORMANCE_RISK, CONFIDENCE, PRIORITY_SCORE)
                    SELECT SIGNAL_ID, ACTION_TYPE, TARGET_OBJECT, PARSE_JSON(CURRENT_CONFIG_STR),
                           PARSE_JSON(PROPOSED_CONFIG_STR), PROPOSED_SQL, EST_MONTHLY_SAVINGS_CR,
                           PERFORMANCE_RISK, CONFIDENCE, PRIORITY_SCORE
                    FROM NEW_RECS
                """).collect()
            return f"Modeled {len(rows)} recommendations from {len(signals)} signals"
        $$;

4. Validate estimates after application: compare actual 14-day credits against the modeled value and store the variance so confidence factors can be recalibrated.

## Coding Conventions
- Estimates are in credits per month; currency conversion happens only in presentation.
- Every recommendation carries CURRENT_CONFIG so rollback SQL is always derivable.
- Keep reduction factors and risk weights in configuration for tuning per customer.
- Never execute PROPOSED_SQL inside the modeling procedure.