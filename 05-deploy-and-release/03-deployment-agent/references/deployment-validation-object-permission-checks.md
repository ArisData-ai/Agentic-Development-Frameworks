---
name: deployment-validation-object-permission-checks
description: This skill defines Snowpark Python procedures that compare the deployment manifest with the live Snowflake catalog, checking object existence, column names and data types, task and object state, and role grants, and record each deviation in the validation results table with expected and actual evidence. Use this skill for any task involving missing or partially deployed objects, schema drift, column type mismatches, suspended tasks after release, missing or extra grants, RBAC drift, ownership checks, or adding new object or permission checks to post-deployment validation.
---
# Deployment Validation Object and Permission Checks

## Overview
This skill covers the first half of the agent's Act step. Two Snowpark procedures read the `objects` and `grants` sections of the deployment manifest, query the live catalog with `INFORMATION_SCHEMA` and `SHOW` commands, and write one result row per check into `VALIDATION_RESULTS`. They use live catalog queries because `ACCOUNT_USAGE` views can lag by up to several hours, which is too slow for detection within minutes.

## When to Use
- Confirming that every object in a release exists with the expected structure.
- Catching partial deployments where some scripts ran and others failed.
- Detecting grants that are missing, revoked, or unexpectedly widened.
- Adding new object types or permission rules to the validation catalog.

## Instructions

### 1. Object validation procedure
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.VALIDATE_OBJECTS(P_DEPLOYMENT_ID STRING, P_RUN_ID STRING)
    RETURNS STRING
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.11'
    PACKAGES = ('snowflake-snowpark-python')
    HANDLER = 'run'
    EXECUTE AS CALLER
    AS
    $$
    import json

    SEVERITY = {"TABLE": "CRITICAL", "VIEW": "CRITICAL", "DYNAMIC_TABLE": "CRITICAL",
                "PROCEDURE": "HIGH", "FUNCTION": "HIGH", "TASK": "HIGH", "STREAM": "HIGH"}

    def record(session, run_id, dep_id, check, obj, sev, status, expected, actual, details):
        session.sql(
            """INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
               (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, OBJECT_NAME, SEVERITY, STATUS, EXPECTED, ACTUAL, DETAILS)
               SELECT ?, ?, 'OBJECT', ?, ?, ?, ?, PARSE_JSON(?), PARSE_JSON(?), ?""",
            params=[run_id, dep_id, check, obj, sev, status,
                    json.dumps(expected), json.dumps(actual), details]).collect()

    def split(fqn):
        db, sch, name = [p.strip('"').upper() for p in fqn.split(".")]
        return db, sch, name

    def exists(session, otype, db, sch, name):
        if otype in ("TABLE", "VIEW", "DYNAMIC_TABLE"):
            rows = session.sql(
                f"SELECT TABLE_TYPE FROM {db}.INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?",
                params=[sch, name]).collect()
            return len(rows) > 0
        show_map = {"TASK": "TASKS", "STREAM": "STREAMS", "PROCEDURE": "PROCEDURES", "FUNCTION": "USER FUNCTIONS"}
        rows = session.sql(f"SHOW {show_map[otype]} LIKE '{name}' IN SCHEMA {db}.{sch}").collect()
        return len(rows) > 0

    def run(session, p_deployment_id, p_run_id):
        manifest = json.loads(session.sql(
            "SELECT MANIFEST FROM DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG WHERE DEPLOYMENT_ID = ?",
            params=[p_deployment_id]).collect()[0][0])
        checked = 0
        for obj in manifest.get("objects", []):
            otype, fqn = obj["type"].upper(), obj["name"]
            sev = SEVERITY.get(otype, "MEDIUM")
            db, sch, name = split(fqn)
            try:
                found = exists(session, otype, db, sch, name)
                record(session, p_run_id, p_deployment_id, "OBJECT_EXISTS", fqn, sev,
                       "PASS" if found else "FAIL", {"exists": True}, {"exists": found},
                       None if found else f"{otype} {fqn} not found after deployment")
                checked += 1
                if not found:
                    continue

                if obj.get("columns"):
                    actual_cols = {r["COLUMN_NAME"]: r["DATA_TYPE"] for r in session.sql(
                        f"SELECT COLUMN_NAME, DATA_TYPE FROM {db}.INFORMATION_SCHEMA.COLUMNS "
                        f"WHERE TABLE_SCHEMA = ? AND TABLE_NAME = ?", params=[sch, name]).collect()}
                    for col in obj["columns"]:
                        cname, ctype = col["name"].upper(), col["type"].upper()
                        got = actual_cols.get(cname)
                        ok = got is not None and got.startswith(ctype.split("(")[0])
                        record(session, p_run_id, p_deployment_id, "COLUMN_STRUCTURE", f"{fqn}.{cname}", "HIGH",
                               "PASS" if ok else "FAIL", {"type": ctype}, {"type": got},
                               None if ok else f"Column {cname} expected {ctype}, found {got}")
                        checked += 1

                if otype == "TASK" and obj.get("state"):
                    state = session.sql(f"SHOW TASKS LIKE '{name}' IN SCHEMA {db}.{sch}").collect()[0]["state"]
                    ok = state.lower() == obj["state"].lower()
                    record(session, p_run_id, p_deployment_id, "TASK_STATE", fqn, "HIGH",
                           "PASS" if ok else "FAIL", {"state": obj["state"]}, {"state": state},
                           None if ok else f"Task {fqn} is {state}, expected {obj['state']}")
                    checked += 1
            except Exception as e:
                record(session, p_run_id, p_deployment_id, "OBJECT_CHECK_ERROR", fqn, sev, "ERROR",
                       obj, {}, str(e)[:4000])
        return f"Object checks executed: {checked}"
    $$;

### 2. Permission validation procedure
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.VALIDATE_PERMISSIONS(P_DEPLOYMENT_ID STRING, P_RUN_ID STRING)
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
        manifest = json.loads(session.sql(
            "SELECT MANIFEST FROM DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG WHERE DEPLOYMENT_ID = ?",
            params=[p_deployment_id]).collect()[0][0])
        grants = manifest.get("grants", [])
        cache, checked = {}, 0

        for g in grants:
            role, priv = g["role"].upper(), g["privilege"].upper()
            on_type, fqn = g["on"].upper(), g["name"].upper()
            key = (on_type, fqn)
            try:
                if key not in cache:
                    rows = session.sql(f"SHOW GRANTS ON {on_type} {fqn}").collect()
                    cache[key] = {(r["grantee_name"].upper(), r["privilege"].upper()) for r in rows
                                  if r["granted_to"] == "ROLE"}
                ok = (role, priv) in cache[key]
                status, details = ("PASS", None) if ok else ("FAIL", f"{role} lacks {priv} on {on_type} {fqn}")
            except Exception as e:
                ok, status, details = False, "ERROR", str(e)[:4000]

            session.sql(
                """INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                   (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, OBJECT_NAME, SEVERITY, STATUS, EXPECTED, ACTUAL, DETAILS)
                   SELECT ?, ?, 'PERMISSION', 'GRANT_PRESENT', ?, 'HIGH', ?, PARSE_JSON(?), PARSE_JSON(?), ?""",
                params=[p_run_id, p_deployment_id, fqn, status,
                        json.dumps({"role": role, "privilege": priv}),
                        json.dumps({"granted": ok}), details]).collect()
            checked += 1

        # Flag privileges granted to PUBLIC on release objects as unexpected widening
        for (on_type, fqn), held in cache.items():
            public = sorted(p for (r, p) in held if r == "PUBLIC")
            if public:
                session.sql(
                    """INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                       (RUN_ID, DEPLOYMENT_ID, CHECK_CATEGORY, CHECK_NAME, OBJECT_NAME, SEVERITY, STATUS, EXPECTED, ACTUAL, DETAILS)
                       SELECT ?, ?, 'PERMISSION', 'NO_PUBLIC_GRANTS', ?, 'CRITICAL', 'FAIL',
                              PARSE_JSON('{"public_privileges": []}'), PARSE_JSON(?), ?""",
                    params=[p_run_id, p_deployment_id, fqn, json.dumps({"public_privileges": public}),
                            f"PUBLIC holds {public} on {fqn}"]).collect()
        return f"Permission checks executed: {checked}"
    $$;

### 3. Required privileges for the validator role
    GRANT USAGE ON DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT USAGE ON ALL SCHEMAS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT USAGE ON FUTURE SCHEMAS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT MONITOR ON ALL TASKS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT MONITOR ON FUTURE TASKS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT REFERENCES ON ALL TABLES IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT REFERENCES ON FUTURE TABLES IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT REFERENCES ON ALL VIEWS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;
    GRANT REFERENCES ON FUTURE VIEWS IN DATABASE SALES_DB TO ROLE DEPLOY_VALIDATOR_ROLE;

## Best Practices
- Generate expected grants from the same RBAC-as-code source the deployment uses, so the manifest and the grants never diverge.
- Mark missing tables, views, and dynamic tables as `CRITICAL`; a missing consumer-facing object always blocks the release.
- Cache `SHOW GRANTS` output per object to keep runs fast on large releases.
- Grant the validator role only metadata visibility (`USAGE`, `MONITOR`, `REFERENCES`), never `SELECT` on sensitive data or `MANAGE GRANTS`, unless security explicitly approves it.
- Record a check that throws an exception as `ERROR` and keep going, so one bad object does not hide the other findings.

## Coding Conventions
- One procedure per check category; each one takes `(P_DEPLOYMENT_ID, P_RUN_ID)` and returns a summary string.
- Bind every value with `params=[...]`; build identifiers into the SQL only after the manifest has been validated.
- Check names use uppercase snake case (`OBJECT_EXISTS`, `COLUMN_STRUCTURE`, `TASK_STATE`, `GRANT_PRESENT`, `NO_PUBLIC_GRANTS`).
- `EXPECTED` and `ACTUAL` are always JSON objects with matching keys.