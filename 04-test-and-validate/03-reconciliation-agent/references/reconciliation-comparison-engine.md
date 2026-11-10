---
name: reconciliation-comparison-engine
description: This skill defines the Snowpark-based comparison engine that reconciles source and target data in three tiers, row counts, aggregate control totals, and hash-based key-field comparison, and writes per-run results and record-level variances to Snowflake control tables. Use this skill for any task involving writing reconciliation stored procedures, comparing legacy extracts against Snowflake tables, detecting missing, extra, or mismatched records, computing HASH_AGG fingerprints, normalizing data types before comparison, or scaling source-to-target checks across large migration waves.
---

# Reconciliation Comparison Engine

## Overview
This skill implements the COMPARE stage. It executes queued runs from RECON_RUN, applies tiered checks from cheapest to most detailed, and records outcomes in RECON_RESULT and RECON_VARIANCE.

    Tier 1  Row count         source count vs target count, within tolerance
    Tier 2  Control totals    SUM of each measure column, within absolute tolerance
    Tier 3  Key-field compare HASH_AGG fast path; FULL OUTER JOIN on keys only when hashes differ

## When to Use
- Implementing or extending SP_RUN_QUEUED_RECONS.
- Adding field-level reconciliation for high-criticality objects.
- Tuning comparison performance on large tables.

## Instructions

1. Create result and variance tables:

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.RECON_RESULT (
            RUN_ID STRING, CONFIG_ID STRING, CHECK_TYPE STRING,   -- ROW_COUNT, CONTROL_TOTAL, KEY_FIELD
            METRIC STRING, SOURCE_VALUE NUMBER(38,4), TARGET_VALUE NUMBER(38,4),
            DIFFERENCE NUMBER(38,4), STATUS STRING, DETAIL VARIANT,
            CHECKED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.RECON_VARIANCE (
            RUN_ID STRING, CONFIG_ID STRING, RECORD_KEY OBJECT,
            VARIANCE_TYPE STRING,                                 -- MISSING_IN_TARGET, EXTRA_IN_TARGET, VALUE_MISMATCH
            ROOT_CAUSE_HINT STRING, MISMATCHED_COLUMNS ARRAY,
            DETECTED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Normalize before comparing. Build a comparison view per side that trims strings, casts to target types, converts timestamps to UTC, and rounds decimals to the target scale. Compare the views, never raw legacy columns.

3. Implement the engine as a Snowpark Python procedure:

        CREATE OR REPLACE PROCEDURE RECON_DB.CONTROL.SP_RUN_QUEUED_RECONS()
        RETURNS STRING
        LANGUAGE PYTHON
        RUNTIME_VERSION = '3.11'
        PACKAGES = ('snowflake-snowpark-python')
        HANDLER = 'main'
        EXECUTE AS OWNER
        AS
        $$
        import json
        from snowflake.snowpark import functions as F

        CTL = "RECON_DB.CONTROL"

        def record(session, run_id, cfg_id, check, metric, src, tgt, status, detail=None):
            session.sql(f"""
                INSERT INTO {CTL}.RECON_RESULT
                    (RUN_ID, CONFIG_ID, CHECK_TYPE, METRIC, SOURCE_VALUE, TARGET_VALUE, DIFFERENCE, STATUS, DETAIL)
                SELECT ?, ?, ?, ?, ?, ?, ?, ?, PARSE_JSON(?)
            """, params=[run_id, cfg_id, check, metric, src, tgt,
                         None if src is None or tgt is None else tgt - src,
                         status, json.dumps(detail or {})]).collect()

        def within(src, tgt, pct=None, abs_tol=None):
            diff = abs((tgt or 0) - (src or 0))
            if pct is not None:
                return diff <= (abs(src or 0) * pct / 100)
            return diff <= (abs_tol or 0)

        def run_one(session, run):
            cfg = session.table(f"{CTL}.RECON_CONFIG").filter(F.col("CONFIG_ID") == run["CONFIG_ID"]).collect()[0]
            audit = session.table(f"{CTL}.LOAD_AUDIT").filter(F.col("LOAD_ID") == run["LOAD_ID"]).collect()[0]
            keys = json.loads(cfg["KEY_COLUMNS"])
            measures = json.loads(cfg["MEASURE_COLUMNS"] or "[]")
            cols = json.loads(cfg["COMPARE_COLUMNS"] or "[]")
            where = f"WHERE {cfg['FILTER_PREDICATE']}" if cfg["FILTER_PREDICATE"] else ""
            tgt_sql = f"SELECT * FROM {cfg['TARGET_OBJECT']} {where}"
            tgt = session.sql(tgt_sql)
            results = []

            # Tier 1: row count against extraction-time evidence
            src_cnt, tgt_cnt = audit["SOURCE_ROW_COUNT"], tgt.count()
            ok = within(src_cnt, tgt_cnt, pct=float(cfg["ROW_COUNT_TOLERANCE_PCT"]))
            results.append("PASS" if ok else "FAIL")
            record(session, run["RUN_ID"], cfg["CONFIG_ID"], "ROW_COUNT", "ROW_COUNT",
                   src_cnt, tgt_cnt, results[-1])

            # Tier 2: control totals
            src_totals = json.loads(audit["SOURCE_CONTROL_TOTALS"] or "{}")
            if measures:
                tgt_totals = tgt.agg(*[F.sum(F.col(m)).alias(m) for m in measures]).collect()[0].as_dict()
                for m in measures:
                    s, t = src_totals.get(m), float(tgt_totals.get(m) or 0)
                    ok = s is not None and within(s, t, abs_tol=float(cfg["AMOUNT_TOLERANCE_ABS"]))
                    results.append("PASS" if ok else "FAIL")
                    record(session, run["RUN_ID"], cfg["CONFIG_ID"], "CONTROL_TOTAL", m, s, t, results[-1])

            # Tier 3: key-field compare against landed source snapshot
            if cfg["SOURCE_SNAPSHOT_TABLE"] and cols:
                src_sql = f"SELECT * FROM {cfg['SOURCE_SNAPSHOT_TABLE']} {where}"
                col_list, key_list = ", ".join(cols), ", ".join(keys)
                h = session.sql(f"""
                    SELECT (SELECT HASH_AGG({key_list}, {col_list}) FROM ({src_sql})) AS S,
                           (SELECT HASH_AGG({key_list}, {col_list}) FROM ({tgt_sql})) AS T
                """).collect()[0]
                if h["S"] == h["T"]:
                    results.append("PASS")
                    record(session, run["RUN_ID"], cfg["CONFIG_ID"], "KEY_FIELD", "HASH_AGG", None, None, "PASS")
                else:
                    join = " AND ".join([f"S.{k} = T.{k}" for k in keys])
                    key_obj = ", ".join([f"'{k}', COALESCE(S.{k}, T.{k})" for k in keys])
                    n = session.sql(f"""
                        INSERT INTO {CTL}.RECON_VARIANCE (RUN_ID, CONFIG_ID, RECORD_KEY, VARIANCE_TYPE)
                        WITH S AS (SELECT {key_list}, HASH({col_list}) AS RH FROM ({src_sql})),
                             T AS (SELECT {key_list}, HASH({col_list}) AS RH FROM ({tgt_sql}))
                        SELECT ?, ?, OBJECT_CONSTRUCT({key_obj}),
                               CASE WHEN T.RH IS NULL THEN 'MISSING_IN_TARGET'
                                    WHEN S.RH IS NULL THEN 'EXTRA_IN_TARGET'
                                    ELSE 'VALUE_MISMATCH' END
                        FROM S FULL OUTER JOIN T ON {join}
                        WHERE S.RH IS DISTINCT FROM T.RH
                    """, params=[run["RUN_ID"], cfg["CONFIG_ID"]]).collect()[0][0]
                    results.append("FAIL")
                    record(session, run["RUN_ID"], cfg["CONFIG_ID"], "KEY_FIELD", "VARIANCE_ROWS",
                           0, n, "FAIL", {"variance_rows": n})

            return "FAIL" if "FAIL" in results else "PASS"

        def main(session):
            runs = session.table(f"{CTL}.RECON_RUN").filter(F.col("STATUS") == "QUEUED").collect()
            for run in runs:
                session.sql(f"UPDATE {CTL}.RECON_RUN SET STATUS='RUNNING', STARTED_AT=CURRENT_TIMESTAMP() WHERE RUN_ID=?",
                            params=[run["RUN_ID"]]).collect()
                try:
                    status = run_one(session, run)
                except Exception as e:
                    status = "ERROR"
                    record(session, run["RUN_ID"], run["CONFIG_ID"], "ENGINE", "ERROR", None, None, "ERROR", {"error": str(e)})
                session.sql(f"UPDATE {CTL}.RECON_RUN SET STATUS=?, ENDED_AT=CURRENT_TIMESTAMP() WHERE RUN_ID=?",
                            params=[status, run["RUN_ID"]]).collect()
            return f"PROCESSED: {len(runs)}"
        $$;

4. Scale. For tables above several hundred million rows, partition Tier 3 by a date or business-unit column and run partitions in parallel tasks; compare HASH_AGG per partition first and drill down only into partitions that differ.

5. Re-runs. A re-run creates a new RUN_ID linked to the same LOAD_ID; prior results remain for audit.

## Coding Conventions
- Always run Tier 1 and Tier 2 before Tier 3; skip key-level joins when HASH_AGG fingerprints match.
- Validate KEY_COLUMNS, COMPARE_COLUMNS, and FILTER_PREDICATE against INFORMATION_SCHEMA before execution to prevent injection through config values.
- Use bind parameters for every runtime value; interpolate only validated identifiers.
- HASH and HASH_AGG are for equality detection, not security; never expose them as data fingerprints to external parties.
- Exclude non-deterministic or audit columns (load timestamps, surrogate keys generated in Snowflake) from COMPARE_COLUMNS.
- Wrap each run in its own error handling so one failed object does not stop the batch.