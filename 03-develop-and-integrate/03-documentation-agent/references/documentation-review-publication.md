---
name: documentation-review-publication
description: This skill defines the review and publication workflow that routes documentation drafts to the Engineering Team, records corrections and approvals, publishes approved versions, and makes them searchable through Cortex Search. Use this skill for any task involving documentation approval, reviewer workflows, change requests, audit trails, versioning, publishing, object comments, Cortex Search service setup, or searchable documentation for onboarding, support, and audit teams.
---

# Documentation Review and Publication

## Overview

This skill covers the Hand off stage. It keeps the Engineering Team in control of documentation quality and creates a traceable path from draft to published content.

## When to Use

- Setting up Engineering Team review workflows.
- Recording corrections, approvals, or rejections.
- Publishing approved documentation versions.
- Creating or updating Cortex Search services.
- Preparing documentation evidence for audits.

## Instructions

1. Create review and publication tables.

        CREATE TABLE IF NOT EXISTS DOC_AGENT_DB.CORE.DOC_REVIEW_LOG (
            REVIEW_ID STRING DEFAULT UUID_STRING(),
            DRAFT_ID STRING NOT NULL,
            REVIEWER STRING DEFAULT CURRENT_USER(),
            DECISION STRING NOT NULL,
            REVIEW_NOTES STRING,
            REVIEWED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS DOC_AGENT_DB.CORE.DOC_PUBLISHED (
            PUBLISHED_ID STRING DEFAULT UUID_STRING(),
            DRAFT_ID STRING NOT NULL,
            OBJECT_FQN STRING NOT NULL,
            DOC_TYPE STRING NOT NULL,
            DOC_TEXT STRING NOT NULL,
            VERSION_NUMBER NUMBER NOT NULL,
            IS_CURRENT BOOLEAN DEFAULT TRUE,
            APPROVED_BY STRING NOT NULL,
            PUBLISHED_AT TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Give reviewers a complete package.

    - Object name, owner, change type, and source change reference.
    - The generated draft and its source metadata.
    - Changes from the current published version.
    - Open questions and undocumented items.

3. Apply one review decision.

    | Decision | Result |
    | --- | --- |
    | `APPROVED` | Draft is eligible for publication |
    | `CHANGES_REQUESTED` | Draft returns for correction or regeneration |
    | `REJECTED` | Draft closes and is not published |

4. Require an Engineering Team reviewer to resolve or accept every open question before approval.

5. Publish only approved drafts, and retire the previous version in the same transaction.

        BEGIN;

        UPDATE DOC_AGENT_DB.CORE.DOC_PUBLISHED
        SET IS_CURRENT = FALSE
        WHERE OBJECT_FQN = :object_fqn
          AND DOC_TYPE = :doc_type
          AND IS_CURRENT = TRUE;

        INSERT INTO DOC_AGENT_DB.CORE.DOC_PUBLISHED
            (DRAFT_ID, OBJECT_FQN, DOC_TYPE, DOC_TEXT, VERSION_NUMBER, APPROVED_BY)
        SELECT
            d.DRAFT_ID,
            d.OBJECT_FQN,
            d.DOC_TYPE,
            :approved_doc_text,
            :next_version,
            r.REVIEWER
        FROM DOC_AGENT_DB.CORE.DOC_DRAFTS d
        JOIN DOC_AGENT_DB.CORE.DOC_REVIEW_LOG r
            ON r.DRAFT_ID = d.DRAFT_ID
        WHERE d.DRAFT_ID = :draft_id
          AND r.DECISION = 'APPROVED'
        QUALIFY ROW_NUMBER() OVER (ORDER BY r.REVIEWED_AT DESC) = 1;

        UPDATE DOC_AGENT_DB.CORE.DOC_DRAFTS
        SET STATUS = 'PUBLISHED'
        WHERE DRAFT_ID = :draft_id;

        COMMIT;

6. Create the Cortex Search service. Enable change tracking on the source table first.

        ALTER TABLE DOC_AGENT_DB.CORE.DOC_PUBLISHED
            SET CHANGE_TRACKING = TRUE;

        CREATE OR REPLACE CORTEX SEARCH SERVICE DOC_AGENT_DB.CORE.DOC_SEARCH_SERVICE
            ON DOC_TEXT
            ATTRIBUTES OBJECT_FQN, DOC_TYPE
            WAREHOUSE = DOC_AGENT_WH
            TARGET_LAG = '1 hour'
        AS
            SELECT DOC_TEXT, OBJECT_FQN, DOC_TYPE, VERSION_NUMBER, PUBLISHED_AT
            FROM DOC_AGENT_DB.CORE.DOC_PUBLISHED
            WHERE IS_CURRENT = TRUE;

7. Apply approved descriptions to Snowflake objects when the deployment requires it.

        COMMENT ON TABLE <DATABASE>.<SCHEMA>.<TABLE> IS '<Approved description>';

8. Grant search access to documentation consumers.

        GRANT USAGE ON DATABASE DOC_AGENT_DB TO ROLE DOC_READER;
        GRANT USAGE ON SCHEMA DOC_AGENT_DB.CORE TO ROLE DOC_READER;
        GRANT USAGE ON CORTEX SEARCH SERVICE DOC_AGENT_DB.CORE.DOC_SEARCH_SERVICE
            TO ROLE DOC_READER;

## Coding Conventions

- Keep the review log append-only.
- Preserve prior published versions for traceability.
- Make `DOC_REVIEWER` the only role that can record decisions.
- Keep `DOC_AGENT_RUNNER` out of approval and publication permissions.
- Use bind variables for reviewer input and document values.
- Index only approved, current documentation in Cortex Search.