---
name: dq-quarantine-management
description: This skill defines how the Data Quality Agent isolates records that fail blocker or major rules into governed quarantine tables, publishes only passing rows to consumer schemas, and keeps held data recoverable until the QA Lead releases or discards it, built on Snowflake tables, Streams and Tasks, and Horizon access policies. Use this skill for any task involving quarantining bad records, holding failed loads, write-audit-publish patterns, separating clean and failed data, releasing or reprocessing quarantined rows, or protecting finance, retail, and ERP reports from defective data.
---
# DQ Quarantine Management

## Overview
Quarantine keeps defects away from consumers without losing data. The agent follows a write, audit, publish pattern: loads land in a staging layer, checks run, failing records move to quarantine, and only passing records publish. Release and discard decisions belong to the QA Lead.

## When to Use
- Blocking defective rows from reaching reports or downstream models
- Holding an entire load when a BLOCKER rule fails at table level
- Releasing corrected or approved records after QA review
- Auditing what was held, why, and who released it

## Instructions

### 1. Decide record level versus load level
    Row-identifiable failure (null key, invalid status, negative amount) -> quarantine failing rows, publish the rest
    Table-level failure (freshness, volume anomaly, duplicate burst)     -> hold the whole load
    Severity MINOR                                                       -> publish, log, include in escalation
    Severity MAJOR or BLOCKER                                            -> quarantine or hold

### 2. Create quarantine storage
    CREATE TABLE IF NOT EXISTS DQ_DB.QUARANTINE.QUARANTINED_RECORDS (
        QUARANTINE_ID   STRING DEFAULT UUID_STRING(),
        RUN_ID          STRING,
        RULE_ID         STRING,
        TABLE_FQN       STRING,
        RECORD_KEY      STRING,
        RECORD_PAYLOAD  VARIANT,          -- full row as OBJECT_CONSTRUCT(*)
        FAILURE_REASON  STRING,
        STATUS          STRING DEFAULT 'HELD',   -- HELD | RELEASED | DISCARDED | REPROCESSED
        DECIDED_BY      STRING,
        DECIDED_AT      TIMESTAMP_LTZ,
        QUARANTINED_AT  TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE TABLE IF NOT EXISTS DQ_DB.QUARANTINE.HELD_LOADS (
        RUN_ID STRING, TABLE_FQN STRING, REASON STRING,
        STATUS STRING DEFAULT 'HELD', DECIDED_BY STRING, DECIDED_AT TIMESTAMP_LTZ
    );

### 3. Quarantine failing rows
SP_QUARANTINE_FAILURES(RUN_ID) generates one INSERT per failed row-level rule:

    INSERT INTO DQ_DB.QUARANTINE.QUARANTINED_RECORDS
        (RUN_ID, RULE_ID, TABLE_FQN, RECORD_KEY, RECORD_PAYLOAD, FAILURE_REASON)
    SELECT :run_id, :rule_id, 'RAW_DB.ERP.SALES_ORDERS',
           ORDER_ID::STRING, OBJECT_CONSTRUCT(*), 'CUSTOMER_ID is null'
    FROM DQ_DB.OPS.STG_SALES_ORDERS_RUN
    WHERE RUN_ID = :run_id AND CUSTOMER_ID IS NULL;

Verify the inserted count equals FAILED_ROWS in DQ_CHECK_RESULTS; on mismatch, hold the whole load.

### 4. Publish only passing rows
    INSERT INTO ANALYTICS_DB.SALES.SALES_ORDERS
    SELECT s.* EXCLUDE (RUN_ID)
    FROM DQ_DB.OPS.STG_SALES_ORDERS_RUN s
    WHERE s.RUN_ID = :run_id
      AND NOT EXISTS (SELECT 1 FROM DQ_DB.QUARANTINE.QUARANTINED_RECORDS q
                      WHERE q.RUN_ID = s.RUN_ID AND q.RECORD_KEY = s.ORDER_ID::STRING
                        AND q.STATUS = 'HELD')
      AND NOT EXISTS (SELECT 1 FROM DQ_DB.QUARANTINE.HELD_LOADS h
                      WHERE h.RUN_ID = s.RUN_ID AND h.STATUS = 'HELD');

Run the publish inside a transaction with the quarantine insert so consumers never see a partial state.

### 5. Release, discard, or reprocess (QA Lead only)
    CREATE OR REPLACE PROCEDURE DQ_DB.OPS.SP_RELEASE_QUARANTINE(RUN_ID STRING, DECISION STRING, NOTE STRING)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    BEGIN
        IF (NOT IS_ROLE_IN_SESSION('DQ_QA_LEAD')) THEN
            RETURN 'DENIED: QA Lead role required';
        END IF;
        UPDATE DQ_DB.QUARANTINE.QUARANTINED_RECORDS
           SET STATUS = :DECISION, DECIDED_BY = CURRENT_USER(), DECIDED_AT = CURRENT_TIMESTAMP()
         WHERE RUN_ID = :RUN_ID AND STATUS = 'HELD';
        -- RELEASED rows are then published by SP_PUBLISH_RELEASED(RUN_ID)
        RETURN 'OK';
    END;
    $$;

    GRANT USAGE ON PROCEDURE DQ_DB.OPS.SP_RELEASE_QUARANTINE(STRING, STRING, STRING) TO ROLE DQ_QA_LEAD;

### 6. Protect quarantine contents
Quarantined payloads may hold sensitive fields. Apply masking or row access policies inherited from the source classification so only DQ_QA_LEAD and the data owner see unmasked values.

## Coding Conventions
- Never DELETE from source or staging; quarantine copies, then publish filters.
- Every quarantined row carries RUN_ID and RULE_ID for traceability.
- Wrap quarantine and publish in BEGIN ... COMMIT; roll back on any count mismatch.
- Release procedures use EXECUTE AS CALLER and a role check; the agent role has no grant on them.
- Retain DISCARDED records for 90 days (configurable) before purge by a scheduled task owned by DQ_QA_LEAD.