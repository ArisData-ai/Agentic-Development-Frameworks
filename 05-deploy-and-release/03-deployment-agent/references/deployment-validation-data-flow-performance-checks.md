---
name: deployment-validation-data-flow-performance-checks
description: This skill defines Snowpark procedures that confirm post-deployment data flows are healthy by checking task run outcomes, stream staleness, dynamic table refresh status, Snowpipe execution state, and table freshness, and that compare smoke-query runtimes with stored performance baselines. Use this skill for any task involving pipeline health after release, failed or suspended tasks, stale streams, lagging dynamic tables, paused pipes, stale data, performance regressions, warehouse sizing impact, baseline management, or adding data flow and performance checks to deployment validation.
---
# Deployment Validation Data Flow and Performance Checks

## Overview
This skill covers the second half of the agent's Act step. It confirms that the deployed pipelines actually move data and that key queries still meet their performance targets. Data flow checks read the `data_flows` section of the manifest. Performance checks run smoke queries from the `performance` section and compare elapsed time with baselines. Every finding is written to `VALIDATION_RESULTS`.

## When to Use
- Verifying that tasks, streams, dynamic tables, and pipes work after a release.
- Detecting broken data flows that object checks miss, such as an object that exists but never refreshes.
- Catching query performance regressions caused by view rewrites, clustering changes, or warehouse changes.
- Creating or updating performance baselines.

## Instructions

### 1. Performance baseline table
    CREATE TABLE IF NOT EXISTS DEPLOY_VALIDATION_DB.CORE.PERFORMANCE_BASELINES (
        ENVIRONMENT     STRING,
        CHECK_NAME      STRING,
        BASELINE_MS     NUMBER,
        TOLERANCE_PCT   NUMBER DEFAULT 50,
        UPDATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 2. Data flow validation procedure
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.VALIDATE_DATA_FLOWS(P_DEPLOYMENT_ID STRING, P_RUN_ID STRING)
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.11'
    PACKAGES = ('snowflake-snowpark-python')
    HANDLER = 'run'
    EXECUTE AS CALLER
    AS
    $$
    import json

    def record(s, run_id, dep_id, check, obj, sev, status, exp, act, details):
        s.sql("""INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                 (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, OBJECT_NAME, SEVERITY, STATUS, EXPECTED, ACTUAL, DETAILS)
                 SELECT ?, ?, 'DATA_FLOW', ?, ?, ?, ?, PARSE_JSON(?), PARSE_JSON(?), ?""",
              params=[run_id, dep_id, check, obj, sev, status, json.dumps(exp), json.dumps(act), details]).collect()

    def check_task(s, f):
        db, sch, name = f["name"].upper().split(".")
        rows = s.sql(f"""SELECT STATE, COMPLETED_TIME, ERROR_MESSAGE,
                                DATEDIFF('minute', COMPLETED_TIME, CURRENT_TIMESTAMP()) AS MINS_AGO
                         FROM TABLE({db}.INFORMATION_SCHEMA.TASK_HISTORY(
                             SCHEDULED_TIME_RANGE_START => DATEADD('hour', -24, CURRENT_TIMESTAMP()),
                             TASK_NAME => '{name}'))
                         WHERE SCHEMA_NAME = '{sch}' AND STATE IN ('SUCCEEDED', 'FAILED')
                         ORDER BY COMPLETED_TIME DESC LIMIT 1""").collect()
        if not rows:
            return "WARN", {"last_state": None}, "No completed run in last 24 hours"
        r = rows[0]
        limit = f.get("max_minutes_since_success", 60)
        if r["STATE"] == "FAILED":
            return "FAIL", {"last_state": "FAILED", "error": r["ERROR_MESSAGE"]}, "Latest task run failed"
        if r["MINS_AGO"] > limit:
            return "WARN", {"minutes_since_success": r["MINS_AGO"]}, f"Last success older than {limit} minutes"
        return "PASS", {"minutes_since_success": r["MINS_AGO"]}, None

    def check_stream(s, f):
        db, sch, name = f["name"].upper().split(".")
        r = s.sql(f"SHOW STREAMS LIKE '{name}' IN SCHEMA {db}.{sch}").collect()
        if not r:
            return "FAIL", {"exists": False}, "Stream not found"
        stale = str(r[0]["stale"]).lower() == "true"
        return ("FAIL" if stale else "PASS"), {"stale": stale, "mode": r[0]["mode"]}, ("Stream is stale" if stale else None)

    def check_dynamic_table(s, f):
        db = f["name"].upper().split(".")[0]
        rows = s.sql(f"""SELECT STATE, STATE_MESSAGE, REFRESH_END_TIME,
                                DATEDIFF('minute', REFRESH_END_TIME, CURRENT_TIMESTAMP()) AS MINS_AGO
                         FROM TABLE({db}.INFORMATION_SCHEMA.DYNAMIC_TABLE_REFRESH_HISTORY(NAME => '{f["name"].upper()}'))
                         ORDER BY REFRESH_END_TIME DESC NULLS LAST LIMIT 1""").collect()
        if not rows:
            return "WARN", {"state": None}, "No refresh history yet"
        r = rows[0]
        if r["STATE"] in ("FAILED", "UPSTREAM_FAILED"):
            return "FAIL", {"state": r["STATE"], "message": r["STATE_MESSAGE"]}, "Dynamic table refresh failed"
        lag = f.get("max_lag_minutes", 60)
        if r["MINS_AGO"] is not None and r["MINS_AGO"] > lag:
            return "WARN", {"minutes_since_refresh": r["MINS_AGO"]}, f"Refresh older than {lag} minutes"
        return "PASS", {"state": r["STATE"], "minutes_since_refresh": r["MINS_AGO"]}, None

    def check_pipe(s, f):
        status = json.loads(s.sql(f"SELECT SYSTEM$PIPE_STATUS('{f['name']}')").collect()[0][0])
        state = status.get("executionState")
        return ("PASS" if state == "RUNNING" else "FAIL"), {"executionState": state}, \
               (None if state == "RUNNING" else f"Pipe state is {state}")

    def check_freshness(s, f):
        r = s.sql(f"""SELECT DATEDIFF('minute', MAX({f['ts_column']}), CURRENT_TIMESTAMP()) AS MINS
                      FROM {f['name']}""").collect()[0]
        mins, limit = r["MINS"], f.get("max_minutes", 60)
        if mins is None:
            return "FAIL", {"minutes_stale": None}, "Table is empty"
        return ("PASS" if mins <= limit else "FAIL"), {"minutes_stale": mins}, \
               (None if mins <= limit else f"Data is {mins} minutes old, limit {limit}")

    HANDLERS = {"TASK": (check_task, "HIGH"), "STREAM": (check_stream, "HIGH"),
                "DYNAMIC_TABLE": (check_dynamic_table, "HIGH"), "PIPE": (check_pipe, "HIGH"),
                "FRESHNESS": (check_freshness, "CRITICAL")}

    def run(session, p_deployment_id, p_run_id):
        manifest = json.loads(session.sql(
            "SELECT MANIFEST FROM DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG WHERE DEPLOYMENT_ID = ?",
            params=[p_deployment_id]).collect()[0][0])
        n = 0
        for f in manifest.get("data_flows", []):
            fn, sev = HANDLERS[f["type"].upper()]
            try:
                status, actual, details = fn(session, f)
            except Exception as e:
                status, actual, details = "ERROR", {}, str(e)[:4000]
            record(session, p_run_id, p_deployment_id, f"{f['type'].upper()}_HEALTH", f["name"],
                   sev, status, f, actual, details)
            n += 1
        return f"Data flow checks executed: {n}"
    $$;

### 3. Performance validation procedure
Use caller's rights so the procedure can turn off the result cache and tag its queries.

    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.VALIDATE_PERFORMANCE(P_DEPLOYMENT_ID STRING, P_RUN_ID STRING)
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.11'
    PACKAGES = ('snowflake-snowpark-python')
    HANDLER = 'run'
    EXECUTE AS CALLER
    AS
    $$
    import json

    def run(session, p_deployment_id, p_run_id):
        row = session.sql("""SELECT MANIFEST, ENVIRONMENT FROM DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG
                             WHERE DEPLOYMENT_ID = ?""", params=[p_deployment_id]).collect()[0]
        manifest, env = json.loads(row[0]), row[1]
        session.sql("ALTER SESSION SET USE_CACHED_RESULT = FALSE").collect()
        session.sql(f"ALTER SESSION SET QUERY_TAG = 'deploy_validation:{p_run_id}'").collect()
        n = 0
        for p in manifest.get("performance", []):
            try:
                b = session.sql("""SELECT BASELINE_MS, TOLERANCE_PCT FROM DEPLOY_VALIDATION_DB.CORE.PERFORMANCE_BASELINES
                                   WHERE ENVIRONMENT = ? AND CHECK_NAME = ?""", params=[env, p["name"]]).collect()
                baseline = b[0]["BASELINE_MS"] if b else p["baseline_ms"]
                tol = b[0]["TOLERANCE_PCT"] if b else p.get("tolerance_pct", 50)

                session.sql(p["sql"]).collect()
                qid = session.sql("SELECT LAST_QUERY_ID()").collect()[0][0]
                elapsed = session.sql("""SELECT TOTAL_ELAPSED_TIME FROM TABLE(INFORMATION_SCHEMA.QUERY_HISTORY_BY_SESSION())
                                         WHERE QUERY_ID = ?""", params=[qid]).collect()[0][0]
                limit = baseline * (1 + tol / 100.0)
                status = "PASS" if elapsed <= limit else ("WARN" if elapsed <= limit * 1.5 else "FAIL")
                details = None if status == "PASS" else f"{elapsed} ms vs baseline {baseline} ms (+{tol}% tolerance)"
                exp, act = {"baseline_ms": baseline, "limit_ms": limit}, {"elapsed_ms": elapsed, "query_id": qid}
            except Exception as e:
                status, details, exp, act = "ERROR", str(e)[:4000], p, {}
            session.sql("""INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                           (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, OBJECT_NAME, SEVERITY, STATUS, EXPECTED, ACTUAL, DETAILS)
                           SELECT ?, ?, 'PERFORMANCE', 'QUERY_RUNTIME', ?, 'MEDIUM', ?, PARSE_JSON(?), PARSE_JSON(?), ?""",
                        params=[p_run_id, p_deployment_id, p["name"], status,
                                json.dumps(exp), json.dumps(act), details]).collect()
            n += 1
        session.sql("ALTER SESSION UNSET QUERY_TAG").collect()
        return f"Performance checks executed: {n}"
    $$;

### 4. Baseline refresh after an approved release
    MERGE INTO DEPLOY_VALIDATION_DB.CORE.PERFORMANCE_BASELINES t
    USING (
        SELECT d.ENVIRONMENT, r.OBJECT_NAME AS CHECK_NAME, r.ACTUAL:elapsed_ms::NUMBER AS ELAPSED_MS
        FROM DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS r
        JOIN DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG d USING (DEPLOYMENT_ID)
        WHERE r.RUN_ID = '<approved_run_id>' AND r.CHECK_CATEGORY = 'PERFORMANCE' AND r.STATUS = 'PASS'
    ) s
    ON t.ENVIRONMENT = s.ENVIRONMENT AND t.CHECK_NAME = s.CHECK_NAME
    WHEN MATCHED THEN UPDATE SET BASELINE_MS = ROUND((t.BASELINE_MS * 0.7) + (s.ELAPSED_MS * 0.3)), UPDATED_AT = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN INSERT (ENVIRONMENT, CHECK_NAME, BASELINE_MS) VALUES (s.ENVIRONMENT, s.CHECK_NAME, s.ELAPSED_MS);

## Best Practices
- Treat freshness failures on consumer-facing tables as `CRITICAL`, since they cause immediate reporting impact.
- Expect the first runs after a deployment to return `WARN` for "no history yet"; schedule a second validation pass 15 to 30 minutes after go-live for flows that run on longer schedules.
- Keep smoke queries light, bounded, and read-only; never run DML or full-table scans on very large tables.
- Update baselines only from runs the Release Gatekeeper approved, using a weighted average so one slow run does not shift the baseline.
- Run performance checks on the same warehouse size that production consumers use.

## Coding Conventions
- Data flow check names use the `<TYPE>_HEALTH` pattern; performance checks use `QUERY_RUNTIME`.
- Map each data flow type to a handler function in a single `HANDLERS` dictionary; add new types there.
- Store the Snowflake `QUERY_ID` in `ACTUAL` for every performance check so results can be traced.
- Tag all validation queries with `deploy_validation:<run_id>` so they can be found in query history.