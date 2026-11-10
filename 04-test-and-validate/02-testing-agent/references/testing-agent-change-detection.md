---
name: testing-agent-change-detection
description: This skill defines the sense phase of the Snowflake Testing Agent, which watches a configured scope of databases for new or changed tables, views, dynamic tables, stored procedures, and pipelines, accepts change notices from CI and Git, expands each change to its downstream dependents, and writes deduplicated change events through ACCOUNT_USAGE, INFORMATION_SCHEMA, OBJECT_DEPENDENCIES, and scheduled Tasks. Use this skill for any task involving detecting schema or DDL changes, tracking LAST_DDL watermarks, registering merged pull requests or deployment events, finding objects impacted by a change, configuring which schemas the agent monitors, or troubleshooting missed or duplicate change events.
---
# Testing Agent: Change Detection

## Overview
Change detection is the agent's trigger. It turns object changes into rows in TESTING_AGENT_DB.TESTS.CHANGE_EVENTS, and every later phase reads from that table. Changes come from three sources:

- **DDL_METADATA:** Object definition changes found through LAST_DDL and LAST_ALTERED.
- **CI_REGISTRATION:** Merged changes pushed by the CI/CD pipeline or the CI/CD Agent.
- **IMPACT_EXPANSION:** Downstream dependents of changed objects, added so regression tests run on them.

## When to Use
- Defining or changing which databases, schemas, and object types the agent monitors.
- Building or tuning the DETECT_CHANGES procedure.
- Connecting CI pipelines or Git repositories to the agent.
- Finding which downstream objects need regression tests after a change.
- Investigating missed, late, or duplicate change events.

## Instructions

### 1. Configure the monitored scope

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.CONFIG.MONITORED_SCOPE (
        DATABASE_NAME   STRING NOT NULL,
        SCHEMA_NAME     STRING NOT NULL,          -- '%' for all schemas
        OBJECT_TYPE     STRING DEFAULT '%',       -- TABLE | VIEW | DYNAMIC_TABLE | PROCEDURE | %
        CRITICALITY     STRING DEFAULT 'MEDIUM',  -- HIGH | MEDIUM | LOW
        ENABLED         BOOLEAN DEFAULT TRUE,
        UPDATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.CONFIG.WATERMARKS (
        SOURCE_NAME     STRING PRIMARY KEY,
        LAST_SEEN_AT    TIMESTAMP_LTZ
    );

### 2. Create the change event store

    CREATE TABLE IF NOT EXISTS TESTING_AGENT_DB.TESTS.CHANGE_EVENTS (
        EVENT_ID        STRING DEFAULT UUID_STRING(),
        OBJECT_FQN      STRING NOT NULL,
        OBJECT_TYPE     STRING NOT NULL,
        CHANGE_TYPE     STRING NOT NULL,           -- CREATED | ALTERED | IMPACTED
        CHANGE_SOURCE   STRING NOT NULL,           -- DDL_METADATA | CI_REGISTRATION | IMPACT_EXPANSION
        CHANGED_AT      TIMESTAMP_LTZ NOT NULL,
        PARENT_EVENT_ID STRING,                    -- set for IMPACTED events
        CRITICALITY     STRING,
        SOURCE_REF      VARIANT,                   -- commit hash, PR id, query id
        STATUS          STRING DEFAULT 'PENDING',  -- PENDING | PROCESSED | IGNORED
        DETECTED_AT     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE STREAM IF NOT EXISTS TESTING_AGENT_DB.TESTS.CHANGE_EVENTS_STREAM
        ON TABLE TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
        APPEND_ONLY = TRUE;

### 3. Detect DDL changes from metadata
ACCOUNT_USAGE can lag by up to about 3 hours. For near-real-time detection on HIGH-criticality scopes, run the same logic against <DB>.INFORMATION_SCHEMA.TABLES.

    MERGE INTO TESTING_AGENT_DB.TESTS.CHANGE_EVENTS t
    USING (
        SELECT
            tb.TABLE_CATALOG || '.' || tb.TABLE_SCHEMA || '.' || tb.TABLE_NAME AS OBJECT_FQN,
            CASE WHEN tb.IS_DYNAMIC = 'YES' THEN 'DYNAMIC_TABLE'
                 WHEN tb.TABLE_TYPE = 'VIEW' THEN 'VIEW'
                 ELSE 'TABLE' END                                              AS OBJECT_TYPE,
            IFF(tb.CREATED = tb.LAST_DDL, 'CREATED', 'ALTERED')                AS CHANGE_TYPE,
            tb.LAST_DDL                                                        AS CHANGED_AT,
            s.CRITICALITY
        FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES tb
        JOIN TESTING_AGENT_DB.CONFIG.MONITORED_SCOPE s
          ON tb.TABLE_CATALOG = s.DATABASE_NAME
         AND tb.TABLE_SCHEMA LIKE s.SCHEMA_NAME
         AND s.ENABLED
        WHERE tb.DELETED IS NULL
          AND tb.LAST_DDL > (SELECT COALESCE(MAX(LAST_SEEN_AT), '1970-01-01'::TIMESTAMP_LTZ)
                             FROM TESTING_AGENT_DB.CONFIG.WATERMARKS
                             WHERE SOURCE_NAME = 'DDL_METADATA')
    ) src
    ON t.OBJECT_FQN = src.OBJECT_FQN AND t.CHANGED_AT = src.CHANGED_AT
    WHEN NOT MATCHED THEN INSERT
        (OBJECT_FQN, OBJECT_TYPE, CHANGE_TYPE, CHANGE_SOURCE, CHANGED_AT, CRITICALITY)
    VALUES
        (src.OBJECT_FQN, src.OBJECT_TYPE, src.CHANGE_TYPE, 'DDL_METADATA', src.CHANGED_AT, src.CRITICALITY);

Repeat for procedures with SNOWFLAKE.ACCOUNT_USAGE.PROCEDURES on LAST_ALTERED, then move the watermark forward:

    MERGE INTO TESTING_AGENT_DB.CONFIG.WATERMARKS w
    USING (SELECT 'DDL_METADATA' AS SOURCE_NAME,
                  MAX(CHANGED_AT) AS LAST_SEEN_AT
           FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
           WHERE CHANGE_SOURCE = 'DDL_METADATA') s
    ON w.SOURCE_NAME = s.SOURCE_NAME
    WHEN MATCHED THEN UPDATE SET w.LAST_SEEN_AT = s.LAST_SEEN_AT
    WHEN NOT MATCHED THEN INSERT VALUES (s.SOURCE_NAME, s.LAST_SEEN_AT);

### 4. Accept CI and Git change notices
The CI pipeline calls this procedure after a merge, once per changed object.

    CREATE OR REPLACE PROCEDURE TESTING_AGENT_DB.ORCH.REGISTER_CHANGE(
        OBJECT_FQN STRING, OBJECT_TYPE STRING, COMMIT_HASH STRING, PR_ID STRING)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS OWNER
    AS
    $$
    BEGIN
        INSERT INTO TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
            (OBJECT_FQN, OBJECT_TYPE, CHANGE_TYPE, CHANGE_SOURCE, CHANGED_AT, SOURCE_REF)
        SELECT UPPER(:OBJECT_FQN), UPPER(:OBJECT_TYPE), 'ALTERED', 'CI_REGISTRATION',
               CURRENT_TIMESTAMP(),
               OBJECT_CONSTRUCT('commit', :COMMIT_HASH, 'pr', :PR_ID);
        RETURN 'REGISTERED';
    END;
    $$;

    GRANT USAGE ON PROCEDURE TESTING_AGENT_DB.ORCH.REGISTER_CHANGE(STRING, STRING, STRING, STRING)
        TO ROLE <CI_SERVICE_ROLE>;

For repositories linked through a Snowflake Git integration, run ALTER GIT REPOSITORY <repo> FETCH on schedule, compare the branch commit hash with the last stored hash, and call REGISTER_CHANGE for each changed object.

### 5. Expand to downstream dependents

    INSERT INTO TESTING_AGENT_DB.TESTS.CHANGE_EVENTS
        (OBJECT_FQN, OBJECT_TYPE, CHANGE_TYPE, CHANGE_SOURCE, CHANGED_AT, PARENT_EVENT_ID, CRITICALITY)
    SELECT DISTINCT
        d.REFERENCING_DATABASE || '.' || d.REFERENCING_SCHEMA || '.' || d.REFERENCING_OBJECT_NAME,
        d.REFERENCING_OBJECT_DOMAIN,
        'IMPACTED',
        'IMPACT_EXPANSION',
        e.CHANGED_AT,
        e.EVENT_ID,
        e.CRITICALITY
    FROM TESTING_AGENT_DB.TESTS.CHANGE_EVENTS e
    JOIN SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES d
      ON e.OBJECT_FQN = d.REFERENCED_DATABASE || '.' || d.REFERENCED_SCHEMA || '.' || d.REFERENCED_OBJECT_NAME
    WHERE e.STATUS = 'PENDING'
      AND e.CHANGE_TYPE <> 'IMPACTED';

Limit expansion to one level by default. Allow deeper levels only for HIGH-criticality scopes.

### 6. Filter noise
- Mark as IGNORED any events on transient, temporary, or agent-owned objects (TESTING_AGENT_DB.%).
- Collapse multiple events for the same OBJECT_FQN within 10 minutes into one PENDING event.
- Do not treat a dynamic table refresh as a change. Only treat a definition change (LAST_DDL) as one.

## Coding Conventions
- Store OBJECT_FQN in uppercase as DATABASE.SCHEMA.OBJECT, without quotes.
- Keep all detection writes as MERGE or guarded INSERT so reruns never create duplicates.
- Store one watermark per CHANGE_SOURCE. Never share watermarks between sources.
- Keep the detection procedure read-only on everything outside TESTING_AGENT_DB.
- Set QUERY_TAG phase to "sense".