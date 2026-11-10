---
name: testing-agent-test-generation
description: This skill defines how the Snowflake Testing Agent turns pending change events into runnable edge-case and regression tests, combining deterministic templates for nulls, boundaries, duplicates, referential integrity, empty inputs, precision, time zones, late-arriving data, and schema contracts with Cortex COMPLETE-generated logic tests, Time Travel baselines, and EXPLAIN-based SQL validation stored in a versioned test catalog. Use this skill for any task involving writing or generating test cases, defining the test contract, adding edge-case categories, building regression baselines for tables, views, dynamic tables, or stored procedures, prompting Cortex to propose tests, validating generated SQL, or managing the PROPOSED, ACTIVE, and RETIRED test lifecycle.
---
# Testing Agent: Test Generation

## Overview
Test generation reads PENDING rows from CHANGE_EVENTS and writes test definitions to TESTING_AGENT_DB.TESTS.TEST_CASES. Every test follows one contract:

**A test is a single SELECT statement that returns only violating rows. Zero rows means PASS.**

Tests come from three generators, applied in this order:

1. **Template generator:** Deterministic edge-case checks built from column metadata.
2. **Regression generator:** Before-and-after comparisons against a known-good baseline.
3. **Cortex generator:** Logic-specific edge cases inferred from the object definition.

## When to Use
- Generating tests for a new or changed object.
- Adding or tuning an edge-case category template.
- Building regression baselines and diff tests.
- Changing the Cortex prompt, model, or output schema.
- Managing test status, versioning, and retirement.

## Instructions

### 1. Create the test catalog

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.TESTS.TEST_CASES (
        TEST_ID         STRING DEFAULT UUID_STRING(),
        TEST_NAME       STRING NOT NULL,
        TARGET_FQN      STRING NOT NULL,
        TEST_KIND       STRING NOT NULL,          -- EDGE_CASE | REGRESSION
        CATEGORY        STRING NOT NULL,          -- see category table
        TEST_SQL        STRING NOT NULL,
        SEVERITY        STRING DEFAULT 'MEDIUM',  -- CRITICAL | HIGH | MEDIUM | LOW
        GENERATOR       STRING NOT NULL,          -- TEMPLATE | REGRESSION | CORTEX
        RATIONALE       STRING,
        SOURCE_EVENT_ID STRING,
        VERSION         NUMBER DEFAULT 1,
        STATUS          STRING DEFAULT 'PROPOSED',-- PROPOSED | ACTIVE | RETIRED
        FALSE_POSITIVES NUMBER DEFAULT 0,
        CREATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        APPROVED_BY     STRING,
        APPROVED_AT     TIMESTAMP_LTZ
    );

### 2. Apply edge-case category templates

    CATEGORY          TRIGGER                          TEMPLATE (violating rows)
    NULL_HANDLING     NOT NULL or key columns          SELECT * FROM {t} WHERE {col} IS NULL
    DUPLICATE_KEY     PK, unique, or *_ID columns      SELECT {key}, COUNT(*) c FROM {t} GROUP BY {key} HAVING c > 1
    BOUNDARY          NUMBER and DATE columns          SELECT * FROM {t} WHERE {col} < {min} OR {col} > {max}
    REFERENTIAL       FK or *_ID matching a dim        SELECT f.* FROM {t} f LEFT JOIN {dim} d ON f.{k}=d.{k} WHERE d.{k} IS NULL AND f.{k} IS NOT NULL
    EMPTY_INPUT       Pipelines and dynamic tables     SELECT 1 WHERE (SELECT COUNT(*) FROM {t}) = 0
    TYPE_PRECISION    NUMBER(p,s) and FLOAT money      SELECT * FROM {t} WHERE {col} <> ROUND({col}, {scale})
    TIMEZONE          TIMESTAMP_NTZ or mixed TZ        SELECT * FROM {t} WHERE {ts_col} > CURRENT_TIMESTAMP()::TIMESTAMP_NTZ
    LATE_ARRIVING     Incremental loads                SELECT * FROM {src} s WHERE s.{ts} < {watermark} AND NOT EXISTS (SELECT 1 FROM {t} x WHERE x.{k}=s.{k})
    SCHEMA_CONTRACT   Any ALTERED event                Compare INFORMATION_SCHEMA.COLUMNS to the CONTRACT_COLUMNS snapshot

Take boundary values from profiling (MIN, MAX, and APPROX_PERCENTILE at 0.001 and 0.999) on the last approved baseline, not from the current data.

### 3. Generate regression tests
Use Time Travel for tables and dynamic tables. For views and procedure outputs, capture a snapshot table at approval time.

    -- Rows present in the baseline but missing or changed now
    SELECT 'MISSING_OR_CHANGED' AS DIFF_TYPE, b.*
    FROM (SELECT * FROM {t} AT(TIMESTAMP => '{baseline_ts}'::TIMESTAMP_LTZ)) b
    EXCEPT
    SELECT 'MISSING_OR_CHANGED', c.* FROM {t} c
    UNION ALL
    -- Aggregate drift beyond tolerance
    SELECT 'AGG_DRIFT', NULL /* pad to column count */
    WHERE ABS((SELECT SUM({measure}) FROM {t})
            - (SELECT SUM({measure}) FROM {t} AT(TIMESTAMP => '{baseline_ts}'::TIMESTAMP_LTZ)))
          > {tolerance_pct} * NULLIF(ABS((SELECT SUM({measure}) FROM {t} AT(TIMESTAMP => '{baseline_ts}'::TIMESTAMP_LTZ))), 0);

    -- Snapshot baseline for views and procedure outputs
    CREATE OR REPLACE TABLE TESTING_AGENT_DB.TESTS.BASELINE_{OBJECT_KEY}
        COMMENT = 'Testing Agent regression baseline'
    AS SELECT * FROM {view_or_output};

For large tables, compare HASH_AGG(*) grouped by partition key rather than running a row-level EXCEPT.

### 4. Generate logic tests with Cortex
Build the prompt inside the GENERATE_TESTS Snowpark procedure from GET_DDL output, column metadata, and the existing test names, so the model does not create duplicates.

    SELECT SNOWFLAKE.CORTEX.COMPLETE(
        (SELECT MODEL_NAME FROM TESTING_AGENT_DB.CONFIG.AGENT_SETTINGS WHERE SETTING = 'TESTGEN_MODEL'),
        [
            {'role': 'system', 'content':
             'You are a Snowflake test engineer. Return ONLY a JSON array. Each item has keys: '
             || 'test_name, category, severity, rationale, test_sql. test_sql must be a single '
             || 'Snowflake SELECT that returns only violating rows. No DML, DDL, or CALL.'},
            {'role': 'user', 'content':
             'Object: ' || :target_fqn || '\nDefinition:\n' || :ddl_text
             || '\nColumns:\n' || :column_json
             || '\nExisting tests:\n' || :existing_test_names
             || '\nPropose up to 8 edge-case tests that the template categories do not cover.'}
        ],
        {'temperature': 0, 'max_tokens': 4000}
    ) AS RESPONSE;

Parse the response with TRY_PARSE_JSON(RESPONSE:choices[0]:messages::STRING). Discard the response if it does not parse.

### 5. Validate before storing
Reject any generated test that fails a check:

    1. Statement starts with SELECT or WITH (case-insensitive, after trimming comments)
    2. Contains no INSERT, UPDATE, DELETE, MERGE, CREATE, ALTER, DROP, TRUNCATE, GRANT, CALL, or PUT
    3. EXPLAIN USING TEXT <test_sql> succeeds under TESTING_AGENT_ROLE
    4. References TARGET_FQN or a declared dependency
    5. Does not duplicate an existing ACTIVE test (normalized SQL hash)

    -- Normalized hash for deduplication
    SHA2(REGEXP_REPLACE(LOWER(TRIM(:test_sql)), '\\s+', ' '), 256)

### 6. Manage the lifecycle
- New tests start as PROPOSED. They run in candidate mode, and their failures appear in the QA queue labeled as candidates.
- The QA Lead promotes a test to ACTIVE, which sets APPROVED_BY and APPROVED_AT and adds it to FULL_REGRESSION.
- To change an ACTIVE test, insert a new VERSION and retire the earlier one. Never update TEST_SQL in place.
- Flag any test whose FALSE_POSITIVES reaches the configured threshold so the QA Lead can review it for retirement.
- When generation is done, mark the source CHANGE_EVENTS rows as PROCESSED.

## Coding Conventions
- Name tests as <CATEGORY>__<OBJECT>__<COLUMN_OR_RULE>, in uppercase with a double underscore between parts.
- Put placeholders in braces ({t}, {col}) and substitute them only in Snowpark. Never concatenate untrusted text into executable SQL without the validation step.
- Use temperature 0 for Cortex and pin the model in CONFIG.AGENT_SETTINGS so results are reproducible.
- Always store RATIONALE. The QA Lead reviews why a test exists, not just what it checks.
- Set QUERY_TAG phase to "generate".