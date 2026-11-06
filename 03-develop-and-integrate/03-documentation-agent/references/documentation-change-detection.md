---
name: documentation-change-detection
description: This skill defines the detection workflow that identifies new or changed pipelines, models, and code, records them in a documentation queue, and flags undocumented objects through account metadata, Streams, and Tasks. Use this skill for any task involving documentation triggers, schema change detection, DDL monitoring, Git or CI/CD change events, queue design, stream-based task scheduling, deduplication, or undocumented object reports.
---

# Documentation Change Detection

## Overview

This skill covers the Sense stage. It converts platform and code changes into a single, deduplicated queue that starts documentation work.

## When to Use

- Setting up documentation triggers.
- Capturing DDL changes in Snowflake objects.
- Accepting change events from Git or CI/CD workflows.
- Finding objects with no approved documentation.
- Troubleshooting missed or duplicate documentation runs.

## Instructions

1. Create the change queue.

        CREATE TABLE IF NOT EXISTS DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE (
            CHANGE_ID STRING DEFAULT UUID_STRING(),
            OBJECT_TYPE STRING NOT NULL,
            OBJECT_FQN STRING NOT NULL,
            CHANGE_TYPE STRING NOT NULL,
            CHANGE_MARKER STRING NOT NULL,
            SOURCE_SYSTEM STRING NOT NULL,
            STATUS STRING DEFAULT 'DETECTED',
            DETECTED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Detect Snowflake object changes with a scheduled task. Use `LAST_DDL` because `LAST_ALTERED` can also change after DML. The lookback window allows for `ACCOUNT_USAGE` latency, and the anti-join prevents duplicates.

        CREATE OR REPLACE TASK DOC_AGENT_DB.CORE.DETECT_OBJECT_CHANGES
            WAREHOUSE = DOC_AGENT_WH
            SCHEDULE = '60 MINUTE'
        AS
        INSERT INTO DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE
            (OBJECT_TYPE, OBJECT_FQN, CHANGE_TYPE, CHANGE_MARKER, SOURCE_SYSTEM)
        SELECT
            t.TABLE_TYPE,
            t.TABLE_CATALOG || '.' || t.TABLE_SCHEMA || '.' || t.TABLE_NAME,
            IFF(t.CREATED = t.LAST_DDL, 'CREATED', 'ALTERED'),
            TO_VARCHAR(t.LAST_DDL),
            'SNOWFLAKE'
        FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
        WHERE t.DELETED IS NULL
          AND t.TABLE_CATALOG = '<TARGET_DATABASE>'
          AND t.LAST_DDL >= DATEADD(HOUR, -6, CURRENT_TIMESTAMP())
          AND NOT EXISTS (
              SELECT 1
              FROM DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE q
              WHERE q.OBJECT_FQN = t.TABLE_CATALOG || '.' || t.TABLE_SCHEMA || '.' || t.TABLE_NAME
                AND q.CHANGE_MARKER = TO_VARCHAR(t.LAST_DDL)
          );

3. Have CI/CD record merged code changes in the same queue. Use the commit SHA as `CHANGE_MARKER`.

        INSERT INTO DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE
            (OBJECT_TYPE, OBJECT_FQN, CHANGE_TYPE, CHANGE_MARKER, SOURCE_SYSTEM)
        SELECT 'CODE', '<REPOSITORY>/<FILE_PATH>', 'ALTERED', '<COMMIT_SHA>', 'GIT';

4. Start generation only when new queue rows exist. The called procedure must consume the stream in a DML statement so its offset advances.

        CREATE OR REPLACE STREAM DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE_STREAM
            ON TABLE DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE
            APPEND_ONLY = TRUE;

        CREATE OR REPLACE TASK DOC_AGENT_DB.CORE.START_DOC_GENERATION
            WAREHOUSE = DOC_AGENT_WH
            SCHEDULE = '15 MINUTE'
            WHEN SYSTEM$STREAM_HAS_DATA('DOC_AGENT_DB.CORE.DOC_CHANGE_QUEUE_STREAM')
        AS
            CALL DOC_AGENT_DB.CORE.RUN_DOC_GENERATION();

5. Resume tasks after deployment.

        ALTER TASK DOC_AGENT_DB.CORE.START_DOC_GENERATION RESUME;
        ALTER TASK DOC_AGENT_DB.CORE.DETECT_OBJECT_CHANGES RESUME;

6. Report objects with no published documentation.

        SELECT
            t.TABLE_CATALOG || '.' || t.TABLE_SCHEMA || '.' || t.TABLE_NAME AS OBJECT_FQN,
            t.TABLE_TYPE,
            t.TABLE_OWNER
        FROM SNOWFLAKE.ACCOUNT_USAGE.TABLES t
        LEFT JOIN DOC_AGENT_DB.CORE.DOC_PUBLISHED p
            ON p.OBJECT_FQN = t.TABLE_CATALOG || '.' || t.TABLE_SCHEMA || '.' || t.TABLE_NAME
           AND p.IS_CURRENT = TRUE
        WHERE t.DELETED IS NULL
          AND t.TABLE_CATALOG = '<TARGET_DATABASE>'
          AND p.OBJECT_FQN IS NULL;

## Coding Conventions

- Queue one row per object and change marker.
- Exclude temporary, scratch, and agent-owned objects through a maintained filter.
- Use task and stream names that describe their action.
- Grant `IMPORTED PRIVILEGES` on the `SNOWFLAKE` database only to the required role.
- Log failed detection runs without dropping queued work.s