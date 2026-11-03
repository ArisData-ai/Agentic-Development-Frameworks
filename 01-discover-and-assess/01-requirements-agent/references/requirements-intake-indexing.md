---
name: requirements-intake-indexing
description: This skill defines how the Requirements Agent detects, parses, chunks, and indexes stakeholder notes, tickets, meeting transcripts, and policy documents into searchable intake data through Snowflake stages, Streams and Tasks, and Cortex Search. Use this skill for any task involving requirements intake, document ingestion, transcript or ticket processing, stage monitoring, text chunking, intake automation, or creating or troubleshooting a Cortex Search service.
---

# Requirements Intake and Indexing

## Overview

This skill covers the Sense phase. It detects new business input, normalizes it into a common structure, and makes every passage searchable and traceable to its source.

## When to Use

- Add or configure an intake source.
- Automate detection of new documents.
- Parse and chunk files or ticket content.
- Create or troubleshoot the intake search service.

## Instructions

1. Create the intake stage and stream.

        CREATE STAGE IF NOT EXISTS INTAKE_STAGE
            DIRECTORY = (ENABLE = TRUE)
            ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

        CREATE STREAM IF NOT EXISTS INTAKE_STAGE_STREAM
            ON STAGE INTAKE_STAGE;

2. Refresh the directory table after uploads, or configure automatic refresh where the stage type supports it.
3. Register each input in `INTAKE_DOCUMENT` with its program, source type, author, source date, file hash, and processing status.
4. Use the file hash to skip duplicate content.
5. Parse files with `AI_PARSE_DOCUMENT`. Load structured ticket text directly from its governed source table.
6. Split the text and retain speaker, section, ticket, or page references.

        INSERT INTO INTAKE_CHUNK (
            CHUNK_ID, DOCUMENT_ID, PROGRAM_ID,
            SOURCE_TYPE, CHUNK_INDEX, CHUNK_TEXT
        )
        SELECT
            UUID_STRING(),
            d.DOCUMENT_ID,
            d.PROGRAM_ID,
            d.SOURCE_TYPE,
            c.INDEX,
            c.VALUE::STRING
        FROM INTAKE_DOCUMENT d,
            LATERAL FLATTEN(
                INPUT => SNOWFLAKE.CORTEX.SPLIT_TEXT_RECURSIVE_CHARACTER(
                    d.PARSED_TEXT, 'none', 1500, 200
                )
            ) c
        WHERE d.STATUS = 'PARSED';

7. Create the Cortex Search service.

        CREATE OR REPLACE CORTEX SEARCH SERVICE INTAKE_SEARCH
            ON CHUNK_TEXT
            ATTRIBUTES PROGRAM_ID, SOURCE_TYPE
            WAREHOUSE = REQ_WH
            TARGET_LAG = '1 hour'
            AS
            SELECT CHUNK_ID, DOCUMENT_ID, PROGRAM_ID, SOURCE_TYPE, CHUNK_TEXT
            FROM INTAKE_CHUNK;

8. Schedule processing only when new data exists.

        CREATE OR REPLACE TASK PROCESS_INTAKE_TASK
            WAREHOUSE = REQ_WH
            SCHEDULE = '15 MINUTES'
            WHEN SYSTEM$STREAM_HAS_DATA('INTAKE_STAGE_STREAM')
            AS CALL PROCESS_INTAKE();

9. Mark unreadable or empty files as `FAILED`, retain the reason, and route them for human review.

## Coding Conventions

- Keep raw source files unchanged.
- Use `DOCUMENT_ID` and `CHUNK_ID` as stable lineage keys.
- Tune chunk size and overlap per source type, and keep the values configurable.
- Filter search requests by `PROGRAM_ID`.
- Size `TARGET_LAG` to the program's review cadence and credit budget.