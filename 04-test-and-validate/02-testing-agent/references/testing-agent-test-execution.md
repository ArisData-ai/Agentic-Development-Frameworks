---
name: testing-agent-test-execution
description: This skill defines how the Snowflake Testing Agent runs catalogued tests through a Snowpark Python stored procedure, scoping runs to changed objects, full regression, or a single object, running destructive edge-case fixtures on zero-copy clones, enforcing warehouse timeouts and retries, and classifying each result as PASS, FAIL, or ERROR with sample failing rows in run and result tables. Use this skill for any task involving running test suites, building or debugging the RUN_TEST_SUITE procedure, seeding synthetic edge-case data into clones, handling timeouts and transient errors, attributing test compute cost, or reading test run history and results.
---
# Testing Agent: Test Execution

## Overview
Test execution runs tests from TEST_CASES and records the results. It does three things:

- Picks tests by run scope: CHANGED, FULL_REGRESSION, or OBJECT:<fqn>.
- Runs each test as a count query and a sample query on TESTING_AGENT_WH.
- Writes one TEST_RUNS header row and one TEST_RESULTS row per test.

A failing test is data for the defect-reporting phase. Execution never stops on a single failure.

## When to Use
- Running or scheduling test suites.
- Building, extending, or debugging RUN_TEST_SUITE.
- Testing pipelines with synthetic edge-case rows on clones.
- Tuning timeouts, retries, or warehouse sizing for test runs.
- Querying run history, pass rates, or flaky tests.

## Instructions

### 1. Create result tables

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.RESULTS.TEST_RUNS (
        RUN_ID          STRING PRIMARY KEY,
        RUN_SCOPE       STRING,
        TRIGGERED_BY    STRING,              -- TASK | AGENT | USER
        STARTED_AT      TIMESTAMP_LTZ,
        ENDED_AT        TIMESTAMP_LTZ,
        TOTAL_TESTS     NUMBER,
        PASSED          NUMBER,
        FAILED          NUMBER,
        ERRORED         NUMBER,
        RUN_STATUS      STRING               -- COMPLETED | PARTIAL | ABORTED
    );

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.RESULTS.TEST_RESULTS (
        RESULT_ID       STRING DEFAULT UUID_STRING(),
        RUN_ID          STRING,
        TEST_ID         STRING,
        TEST_VERSION    NUMBER,
        TARGET_FQN      STRING,
        TEST_STATUS     STRING,              -- PASS | FAIL | ERROR
        IS_CANDIDATE    BOOLEAN,             -- TRUE when the test is still PROPOSED
        FAILING_ROWS    NUMBER,
        SAMPLE_ROWS     VARIANT,             -- up to 20 rows
        ERROR_MESSAGE   STRING,
        QUERY_ID        STRING,
        DURATION_MS     NUMBER,
        ATTEMPTS        NUMBER,
        EXECUTED_AT     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 2. Implement the runner

    CREATE OR REPLACE PROCEDURE TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE(RUN_SCOPE STRING)
    RETURNS VARIANT
    LANGUAGE PYTHON
    RUNTIME_VERSION = '3.11'
    PACKAGES = ('snowflake-snowpark-python')
    HANDLER = 'run'
    EXECUTE AS OWNER
    AS
    $$
    import json, time, uuid
    from snowflake.snowpark.exceptions import SnowparkSQLException

    TRANSIENT = ('000604', '000630', '390114')   # cancelled, timeout, token expiry
    MAX_ATTEMPTS = 2
    SAMPLE_LIMIT = 20

    def select_tests(session, scope):
        base = """SELECT TEST_ID, VERSION, TARGET_FQN, TEST_SQL, STATUS
                  FROM TESTING_AGENT_DB.TESTS.TEST_CASES WHERE STATUS IN ('ACTIVE','PROPOSED')"""
        if scope == 'FULL_REGRESSION':
            return session.sql(base + " AND STATUS = 'ACTIVE'").collect()
        if scope.startswith('OBJECT:'):
            return session.sql(base + " AND TARGET_FQN = ?", params=[scope[7:].upper()]).collect()
        return session.sql(base + """ AND (SOURCE_EVENT_ID IN (
                    SELECT EVENT_ID FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
                    WHERE DETECTED_AT > DATEADD('hour', -24, CURRENT_TIMESTAMP()))
                 OR TARGET_FQN IN (
                    SELECT OBJECT_FQN FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
                    WHERE DETECTED_AT > DATEADD('hour', -24, CURRENT_TIMESTAMP())))""").collect()

    def execute_one(session, sql):
        for attempt in range(1, MAX_ATTEMPTS + 1):
            t0 = time.time()
            try:
                cnt_df = session.sql(f"SELECT COUNT(*) AS C FROM ({sql})")
                cnt = cnt_df.collect()[0]['C']
                qid = session.sql("SELECT LAST_QUERY_ID() AS Q").collect()[0]['Q']
                sample = []
                if cnt > 0:
                    rows = session.sql(f"SELECT * FROM ({sql}) LIMIT {SAMPLE_LIMIT}").collect()
                    sample = [r.as_dict() for r in rows]
                return ('FAIL' if cnt else 'PASS', cnt, sample, None, qid,
                        int((time.time() - t0) * 1000), attempt)
            except SnowparkSQLException as e:
                code = str(getattr(e, 'sql_error_code', ''))
                if attempt < MAX_ATTEMPTS and any(code.endswith(c) for c in TRANSIENT):
                    continue
                return ('ERROR', None, [], str(e)[:2000], None,
                        int((time.time() - t0) * 1000), attempt)

    def run(session, run_scope):
        run_id = str(uuid.uuid4())
        session.sql(f"""ALTER SESSION SET QUERY_TAG =
            '{json.dumps({"agent": "testing", "phase": "execute", "run_id": run_id})}'""").collect()
        session.sql("""INSERT INTO TESTING_AGENT_DB.RESULTS.TEST_RUNS
            (RUN_ID, RUN_SCOPE, TRIGGERED_BY, STARTED_AT)
            SELECT ?, ?, 'TASK', CURRENT_TIMESTAMP()""", params=[run_id, run_scope]).collect()

        counts = {'PASS': 0, 'FAIL': 0, 'ERROR': 0}
        for t in select_tests(session, run_scope):
            status, cnt, sample, err, qid, ms, attempts = execute_one(session, t['TEST_SQL'])
            counts[status] += 1
            session.sql("""INSERT INTO TESTING_AGENT_DB.RESULTS.TEST_RESULTS
                (RUN_ID, TEST_ID, TEST_VERSION, TARGET_FQN, TEST_STATUS, IS_CANDIDATE,
                 FAILING_ROWS, SAMPLE_ROWS, ERROR_MESSAGE, QUERY_ID, DURATION_MS, ATTEMPTS)
                SELECT ?, ?, ?, ?, ?, ?, ?, PARSE_JSON(?), ?, ?, ?, ?""",
                params=[run_id, t['TEST_ID'], t['VERSION'], t['TARGET_FQN'], status,
                        t['STATUS'] == 'PROPOSED', cnt, json.dumps(sample, default=str),
                        err, qid, ms, attempts]).collect()

        total = sum(counts.values())
        session.sql("""UPDATE TESTING_AGENT_DB.RESULTS.TEST_RUNS
            SET ENDED_AT = CURRENT_TIMESTAMP(), TOTAL_TESTS = ?, PASSED = ?, FAILED = ?,
                ERRORED = ?, RUN_STATUS = IFF(? > 0, 'PARTIAL', 'COMPLETED')
            WHERE RUN_ID = ?""",
            params=[total, counts['PASS'], counts['FAIL'], counts['ERROR'],
                    counts['ERROR'], run_id]).collect()
        return {'run_id': run_id, 'scope': run_scope, **counts}
    $$;

Note: some owner's-rights procedures do not allow ALTER SESSION. If that applies, set QUERY_TAG on the task or the service user instead.

### 3. Run destructive fixtures on clones
For pipelines, procedures, and dynamic tables, test behavior by injecting synthetic edge-case rows into a clone, never into the source.

    CREATE OR REPLACE DATABASE TESTING_AGENT_SANDBOX_{RUN_KEY}
        CLONE <TARGET_DB>
        COMMENT = 'Testing Agent ephemeral fixture clone';

    INSERT INTO TESTING_AGENT_SANDBOX_{RUN_KEY}.<SCHEMA>.<SOURCE_TABLE> (ORDER_ID, AMOUNT, ORDER_TS)
    VALUES
        (NULL, 10.00, '2026-01-01 00:00:00'),             -- NULL key
        (1001, -0.01, '2026-01-01 00:00:00'),             -- negative boundary
        (1002, 99999999.99, '2026-03-29 02:30:00'),       -- DST gap timestamp
        (1003, 10.00, '1900-01-01 00:00:00'),             -- far-past date
        (1003, 10.00, '1900-01-01 00:00:00');             -- exact duplicate

    CALL TESTING_AGENT_SANDBOX_{RUN_KEY}.<SCHEMA>.<TRANSFORM_PROC>();
    -- Run the tests against the clone, then drop it
    DROP DATABASE IF EXISTS TESTING_AGENT_SANDBOX_{RUN_KEY};

Grant TESTING_AGENT_ROLE ownership only on sandbox clones. Drop each clone in a finally block, plus a nightly sweep for any sandbox older than 24 hours.

### 4. Control cost and runtime

    CREATE OR REPLACE RESOURCE MONITOR TESTING_AGENT_RM
        WITH CREDIT_QUOTA = 50 FREQUENCY = MONTHLY START_TIMESTAMP = IMMEDIATELY
        TRIGGERS ON 80 PERCENT DO NOTIFY
                 ON 100 PERCENT DO SUSPEND;
    ALTER WAREHOUSE TESTING_AGENT_WH SET RESOURCE_MONITOR = TESTING_AGENT_RM;

- Size up to SMALL only when the 95th-percentile test duration goes above 60 seconds.
- Keep STATEMENT_TIMEOUT_IN_SECONDS on the warehouse. A timeout is recorded as ERROR, not FAIL.
- Attribute credits by QUERY_TAG run_id using SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY.

### 5. Detect flaky tests

    SELECT TEST_ID,
           COUNT_IF(TEST_STATUS = 'FAIL') AS FAILS,
           COUNT_IF(TEST_STATUS = 'PASS') AS PASSES
    FROM TESTING_AGENT_DB.RESULTS.TEST_RESULTS
    WHERE EXECUTED_AT > DATEADD('day', -14, CURRENT_TIMESTAMP())
    GROUP BY TEST_ID
    HAVING FAILS > 0 AND PASSES > 0
       AND COUNT(DISTINCT CONDITIONAL_CHANGE_EVENT(TEST_STATUS) OVER (PARTITION BY TEST_ID ORDER BY EXECUTED_AT)) > 3;

If window functions inside HAVING are not supported in your account, compute status flips in a CTE first. Send flaky tests to the QA Lead as test-quality findings, not product defects.

## Coding Conventions
- Always bind parameters with params=[...]. Interpolate only validated TEST_SQL, never values.
- Truncate error messages to 2,000 characters and limit samples to 20 rows. Mask columns tagged as PII before storing samples.
- Use one RUN_ID per invocation and tag every query with it.
- Keep FAIL (assertion violated) and ERROR (test could not run) separate in all reporting.
- Name sandbox clones TESTING_AGENT_SANDBOX_<RUN_KEY> so cleanup jobs can find them.
- Set QUERY_TAG phase to "execute".