---
name: testing-agent-defect-reporting
description: This skill defines the hand-off phase of the Snowflake Testing Agent, which turns failed and errored test results into deduplicated defect records with severity, likely cause, and confidence, links each defect to its change event, queues findings for QA Lead review, records accept, reprioritize, and reject decisions, sends alerts, and exposes pass rates and defect trends through Cortex Analyst, using Cortex COMPLETE, SYSTEM$SEND_EMAIL, and a semantic model. Use this skill for any task involving logging or triaging defects, root-cause hints for failing tests, defect deduplication, building the QA review queue, recording QA Lead decisions, handling false positives, notifying QA or release owners, or reporting test coverage, pass-rate, and defect-trend metrics for release decisions.
---
# Testing Agent: Defect Reporting and QA Hand-off

## Overview
Defect reporting is where the agent hands off. It reads FAIL and ERROR rows from TEST_RESULTS and does the following:

1. Fingerprints and deduplicates each failure into DEFECT_LOG.
2. Assigns severity and a likely cause with a confidence level.
3. Puts the defect in V_QA_REVIEW_QUEUE and notifies the QA Lead.
4. Records the QA Lead's decision (ACCEPTED, REPRIORITIZED, or REJECTED) and feeds it back into test tuning.

The agent proposes and the QA Lead decides. The agent never closes, deletes, or downgrades a defect on its own.

## When to Use
- Building or changing the REPORT_DEFECTS procedure.
- Tuning severity rules or likely-cause analysis.
- Setting up the QA Lead review queue and decision workflow.
- Configuring notifications for new high-severity defects.
- Reporting pass rates, coverage, and defect trends for release owners.

## Instructions

### 1. Create the defect log

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.DEFECTS.DEFECT_LOG (
        DEFECT_ID         STRING DEFAULT UUID_STRING(),
        FINGERPRINT       STRING NOT NULL,
        TEST_ID           STRING,
        FIRST_RUN_ID      STRING,
        LAST_RUN_ID       STRING,
        TARGET_FQN        STRING,
        CATEGORY          STRING,
        DEFECT_TYPE       STRING,               -- PRODUCT_DEFECT | TEST_ERROR | CANDIDATE_FINDING
        AGENT_SEVERITY    STRING,
        QA_PRIORITY       STRING,               -- set by the QA Lead
        FAILING_ROWS      NUMBER,
        SAMPLE_ROWS       VARIANT,
        LIKELY_CAUSE      STRING,
        CAUSE_CONFIDENCE  STRING,               -- HIGH | MEDIUM | LOW
        RELATED_EVENT_ID  STRING,
        OCCURRENCES       NUMBER DEFAULT 1,
        STATUS            STRING DEFAULT 'OPEN',-- OPEN | ACCEPTED | REPRIORITIZED | REJECTED | RESOLVED
        QA_NOTES          STRING,
        REVIEWED_BY       STRING,
        REVIEWED_AT       TIMESTAMP_LTZ,
        FIRST_SEEN_AT     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        LAST_SEEN_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 2. Fingerprint and deduplicate

    MERGE INTO TESTING_AGENT_DB.DEFECTS.DEFECT_LOG d
    USING (
        SELECT
            SHA2(r.TEST_ID || '|' || r.TARGET_FQN || '|' || r.TEST_STATUS, 256) AS FINGERPRINT,
            r.TEST_ID, r.RUN_ID, r.TARGET_FQN, c.CATEGORY,
            CASE WHEN r.TEST_STATUS = 'ERROR' THEN 'TEST_ERROR'
                 WHEN r.IS_CANDIDATE THEN 'CANDIDATE_FINDING'
                 ELSE 'PRODUCT_DEFECT' END AS DEFECT_TYPE,
            r.FAILING_ROWS, r.SAMPLE_ROWS, c.SEVERITY AS BASE_SEVERITY
        FROM TESTING_AGENT_DB.RESULTS.TEST_RESULTS r
        JOIN TESTING_AGENT_DB.TESTS.TEST_CASES c
          ON r.TEST_ID = c.TEST_ID AND r.TEST_VERSION = c.VERSION
        WHERE r.TEST_STATUS IN ('FAIL', 'ERROR')
          AND r.RUN_ID = :run_id
    ) s
    ON d.FINGERPRINT = s.FINGERPRINT AND d.STATUS IN ('OPEN', 'ACCEPTED', 'REPRIORITIZED')
    WHEN MATCHED THEN UPDATE SET
        d.OCCURRENCES  = d.OCCURRENCES + 1,
        d.LAST_RUN_ID  = s.RUN_ID,
        d.FAILING_ROWS = s.FAILING_ROWS,
        d.SAMPLE_ROWS  = s.SAMPLE_ROWS,
        d.LAST_SEEN_AT = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN INSERT
        (FINGERPRINT, TEST_ID, FIRST_RUN_ID, LAST_RUN_ID, TARGET_FQN, CATEGORY,
         DEFECT_TYPE, AGENT_SEVERITY, FAILING_ROWS, SAMPLE_ROWS)
    VALUES
        (s.FINGERPRINT, s.TEST_ID, s.RUN_ID, s.RUN_ID, s.TARGET_FQN, s.CATEGORY,
         s.DEFECT_TYPE, s.BASE_SEVERITY, s.FAILING_ROWS, s.SAMPLE_ROWS);

Leave REJECTED defects closed. If the same fingerprint fails again after a rejection, open a new defect only when FAILING_ROWS has grown by more than the configured threshold.

### 3. Assign severity

    RULE                                                         SEVERITY
    Target scope criticality HIGH and category DUPLICATE_KEY,
        REFERENTIAL, or REGRESSION                               CRITICAL
    Regression AGG_DRIFT above tolerance on a financial measure  CRITICAL
    Failing rows above 1% of the target row count                Raise one level
    CANDIDATE_FINDING                                            Cap at MEDIUM
    TEST_ERROR                                                   LOW (test-quality issue)

Store the agent's severity in AGENT_SEVERITY. The QA Lead's priority goes in QA_PRIORITY. Never overwrite one with the other.

### 4. Determine the likely cause
First correlate the failure with a recent change event. Then ask Cortex for a short hypothesis.

    -- Nearest change on the target or its upstream within 48 hours
    SELECT e.EVENT_ID, e.OBJECT_FQN, e.CHANGE_TYPE, e.SOURCE_REF
    FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS e
    WHERE (e.OBJECT_FQN = :target_fqn
           OR e.EVENT_ID IN (SELECT PARENT_EVENT_ID FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
                             WHERE OBJECT_FQN = :target_fqn))
      AND e.CHANGED_AT BETWEEN DATEADD('hour', -48, :first_seen_at) AND :first_seen_at
    ORDER BY e.CHANGED_AT DESC
    LIMIT 1;

    SELECT SNOWFLAKE.CORTEX.COMPLETE(
        :model_name,
        [
            {'role': 'system', 'content':
             'You diagnose Snowflake data test failures. Reply ONLY as JSON with keys '
             || 'likely_cause (max 60 words), confidence (HIGH|MEDIUM|LOW), suggested_check. '
             || 'Use HIGH only when a recent change clearly explains the failure.'},
            {'role': 'user', 'content':
             'Test: ' || :test_name || '\nRationale: ' || :rationale
             || '\nTest SQL:\n' || :test_sql
             || '\nSample failing rows:\n' || :sample_json
             || '\nRecent related change:\n' || COALESCE(:change_json, 'none found')}
        ],
        {'temperature': 0, 'max_tokens': 600}
    ) AS RESPONSE;

When no related change event exists, set CAUSE_CONFIDENCE to no higher than MEDIUM.

### 5. Build the QA review queue

    CREATE OR REPLACE SECURE VIEW TESTING_AGENT_DB.DEFECTS.V_QA_REVIEW_QUEUE AS
    SELECT DEFECT_ID, TARGET_FQN, CATEGORY, DEFECT_TYPE, AGENT_SEVERITY,
           FAILING_ROWS, OCCURRENCES, LIKELY_CAUSE, CAUSE_CONFIDENCE,
           RELATED_EVENT_ID, SAMPLE_ROWS, FIRST_SEEN_AT, LAST_SEEN_AT
    FROM TESTING_AGENT_DB.DEFECTS.DEFECT_LOG
    WHERE STATUS = 'OPEN'
    ORDER BY DECODE(AGENT_SEVERITY, 'CRITICAL', 1, 'HIGH', 2, 'MEDIUM', 3, 4),
             LAST_SEEN_AT DESC;

    GRANT SELECT ON VIEW TESTING_AGENT_DB.DEFECTS.V_QA_REVIEW_QUEUE TO ROLE QA_LEAD_ROLE;

### 6. Record QA Lead decisions

    CREATE OR REPLACE PROCEDURE TESTING_AGENT_DB.DEFECTS.RECORD_QA_DECISION(
        DEFECT_ID STRING, DECISION STRING, PRIORITY STRING, NOTES STRING)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS OWNER
    AS
    $$
    BEGIN
        IF (:DECISION NOT IN ('ACCEPTED', 'REPRIORITIZED', 'REJECTED')) THEN
            RETURN 'INVALID_DECISION';
        END IF;

        UPDATE TESTING_AGENT_DB.DEFECTS.DEFECT_LOG
        SET STATUS = :DECISION,
            QA_PRIORITY = COALESCE(:PRIORITY, AGENT_SEVERITY),
            QA_NOTES = :NOTES,
            REVIEWED_BY = CURRENT_USER(),
            REVIEWED_AT = CURRENT_TIMESTAMP()
        WHERE DEFECT_ID = :DEFECT_ID;

        -- A rejection counts as a false positive against the test
        IF (:DECISION = 'REJECTED') THEN
            UPDATE TESTING_AGENT_DB.TESTS.TEST_CASES
            SET FALSE_POSITIVES = FALSE_POSITIVES + 1
            WHERE TEST_ID = (SELECT TEST_ID FROM TESTING_AGENT_DB.DEFECTS.DEFECT_LOG
                             WHERE DEFECT_ID = :DEFECT_ID);
        END IF;
        RETURN 'RECORDED';
    END;
    $$;

    GRANT USAGE ON PROCEDURE TESTING_AGENT_DB.DEFECTS.RECORD_QA_DECISION(STRING, STRING, STRING, STRING)
        TO ROLE QA_LEAD_ROLE;

Provide a separate PROMOTE_TEST(TEST_ID) procedure, granted only to QA_LEAD_ROLE, that moves a PROPOSED test to ACTIVE.

### 7. Notify the QA Lead

    CREATE NOTIFICATION INTEGRATION IF NOT EXISTS TESTING_AGENT_EMAIL_INT
        TYPE = EMAIL
        ENABLED = TRUE
        ALLOWED_RECIPIENTS = ('<qa.lead@company.com>');

    CALL SYSTEM$SEND_EMAIL(
        'TESTING_AGENT_EMAIL_INT',
        '<qa.lead@company.com>',
        'Testing Agent: ' || :new_critical || ' new CRITICAL/HIGH defects',
        'Run ' || :run_id || ' logged ' || :new_defects || ' new defects. '
        || 'Review TESTING_AGENT_DB.DEFECTS.V_QA_REVIEW_QUEUE.'
    );

Send only when new CRITICAL or HIGH defects exist, and send at most one email per run.

### 8. Expose metrics through Cortex Analyst
Stage the semantic model at @TESTING_AGENT_DB.CONFIG.SEMANTIC_MODELS/testing_metrics.yaml.

    name: testing_metrics
    tables:
        - name: test_results
          base_table:
              database: TESTING_AGENT_DB
              schema: RESULTS
              table: TEST_RESULTS
          dimensions:
              - name: target_fqn
                expr: TARGET_FQN
                data_type: VARCHAR
              - name: test_status
                expr: TEST_STATUS
                data_type: VARCHAR
          time_dimensions:
              - name: executed_at
                expr: EXECUTED_AT
                data_type: TIMESTAMP_LTZ
          measures:
              - name: total_tests
                expr: COUNT(*)
                data_type: NUMBER
              - name: pass_rate
                expr: COUNT_IF(TEST_STATUS = 'PASS') / NULLIF(COUNT(*), 0)
                data_type: NUMBER
        - name: defects
          base_table:
              database: TESTING_AGENT_DB
              schema: DEFECTS
              table: DEFECT_LOG
          dimensions:
              - name: severity
                expr: COALESCE(QA_PRIORITY, AGENT_SEVERITY)
                data_type: VARCHAR
              - name: status
                expr: STATUS
                data_type: VARCHAR
              - name: category
                expr: CATEGORY
                data_type: VARCHAR
          time_dimensions:
              - name: first_seen_at
                expr: FIRST_SEEN_AT
                data_type: TIMESTAMP_LTZ
          measures:
              - name: open_defects
                expr: COUNT_IF(STATUS IN ('OPEN','ACCEPTED','REPRIORITIZED'))
                data_type: NUMBER
              - name: rejection_rate
                expr: COUNT_IF(STATUS = 'REJECTED') / NULLIF(COUNT_IF(REVIEWED_AT IS NOT NULL), 0)
                data_type: NUMBER

Example questions this model supports: "What was the pass rate by object last week?", "How many critical defects are open?", and "Which categories have the highest rejection rate?"

## Coding Conventions
- Keep AGENT_SEVERITY and QA_PRIORITY as separate columns. The agent writes only AGENT_SEVERITY.
- Allow status changes only through RECORD_QA_DECISION. Never run a direct UPDATE from agent code.
- Base fingerprints on stable identifiers (test, target, outcome type), never on row counts or timestamps.
- Mask PII in SAMPLE_ROWS before writing and serve samples to QA only through secure views.
- Write likely causes as hypotheses with a confidence level. Never state them as confirmed root cause.
- Set QUERY_TAG phase to "handoff".