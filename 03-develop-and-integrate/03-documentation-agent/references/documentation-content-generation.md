---
name: documentation-content-generation
description: This skill defines the generation workflow that turns queued object changes, Snowflake metadata, Horizon lineage, classification tags, and code definitions into draft data dictionaries, lineage summaries, and technical descriptions with Cortex Agents. Use this skill for any task involving documentation generation, data dictionary creation, column descriptions, lineage summaries, pipeline or model explanations, code documentation, prompt design, draft templates, confidence flags, or quality checks before Engineering Team review.
---

# Documentation Content Generation

## Overview

This skill covers the Act stage. It collects trusted context for each queued object, generates structured drafts, and identifies gaps for reviewers.

## When to Use

- Creating data dictionaries or column descriptions.
- Summarizing upstream and downstream lineage.
- Describing pipelines, models, or code.
- Designing agent prompts and output structures.
- Checking whether drafts are ready for review.

## Instructions

1. Create the draft table.

        CREATE TABLE IF NOT EXISTS DOC_AGENT_DB.CORE.DOC_DRAFTS (
            DRAFT_ID STRING DEFAULT UUID_STRING(),
            CHANGE_ID STRING NOT NULL,
            OBJECT_FQN STRING NOT NULL,
            DOC_TYPE STRING NOT NULL,
            DRAFT_CONTENT VARIANT NOT NULL,
            OPEN_QUESTIONS ARRAY,
            STATUS STRING DEFAULT 'DRAFTED',
            CREATED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Collect context from authoritative sources.

        SELECT GET_DDL('TABLE', '<DATABASE>.<SCHEMA>.<TABLE>');

        SELECT COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COMMENT
        FROM <DATABASE>.INFORMATION_SCHEMA.COLUMNS
        WHERE TABLE_SCHEMA = '<SCHEMA>'
          AND TABLE_NAME = '<TABLE>'
        ORDER BY ORDINAL_POSITION;

        SELECT *
        FROM TABLE(
            SNOWFLAKE.CORE.GET_LINEAGE(
                '<DATABASE>.<SCHEMA>.<TABLE>',
                'TABLE',
                'UPSTREAM',
                2
            )
        );

        SELECT *
        FROM TABLE(
            SNOWFLAKE.CORE.GET_LINEAGE(
                '<DATABASE>.<SCHEMA>.<TABLE>',
                'TABLE',
                'DOWNSTREAM',
                2
            )
        );

    Add classification tags, object owners, approved designs, source mappings, and relevant code when available.

3. Generate only the document types the object requires.

    | Document type | Required content |
    | --- | --- |
    | Data dictionary | Columns, data types, nullability, descriptions, classification tags |
    | Lineage summary | Upstream sources, downstream dependencies, notable transformations |
    | Technical description | Purpose, inputs, outputs, logic, schedule, dependencies, ownership |

4. Give the agent these instructions.

        You are drafting technical documentation for Engineering Team review.
        Use only the supplied metadata, lineage, definitions, and code.
        Do not infer business meaning unless the source states it.
        Add unknown or ambiguous items to OPEN_QUESTIONS.
        Do not include row-level data, secrets, or credentials.
        Return valid JSON that matches the supplied schema.

5. Require structured output.

        {
          "object_fqn": "<DATABASE>.<SCHEMA>.<OBJECT>",
          "doc_type": "DATA_DICTIONARY",
          "summary": "<Source-based summary>",
          "columns": [
            {
              "name": "<COLUMN_NAME>",
              "data_type": "<DATA_TYPE>",
              "description": "<Source-based description>",
              "classification": "<TAG_OR_NULL>"
            }
          ],
          "open_questions": ["<Question for reviewer>"]
        }

6. Validate each draft before review.

    - Confirm that names and data types match current metadata.
    - Confirm that lineage statements match returned lineage records.
    - Confirm that every required section is present.
    - Confirm that open questions remain visible to reviewers.
    - Mark a draft as `IN_REVIEW` only after validation passes.

## Coding Conventions

- Store draft content as JSON in `VARIANT`.
- Pass metadata as structured context, not free text, where possible.
- Generate one draft per object and document type.
- Limit lineage depth to what reviewers can use.
- Use metadata instead of sample rows by default.
- Keep prompt templates version-controlled.