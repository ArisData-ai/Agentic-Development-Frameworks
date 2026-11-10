---
name: release-notes-generation
description: This skill defines how the Release Agent drafts structured release notes from the release manifest, commit messages, ticket references, and test evidence, producing audience-specific versions for business stakeholders, engineers, and compliance reviewers with Snowflake Cortex AI_COMPLETE and governed prompt templates. Use this skill for any task involving release notes, changelogs, go-live summaries, stakeholder change announcements, Jira or Azure DevOps ticket summaries, breaking-change callouts, rollback notes, or requests to write, regenerate, or refine notes for a release.
---
# Release Notes Generation

## Overview
This capability turns a validated manifest into clear release notes so business stakeholders know what is changing and when, engineers know how it deploys, and compliance teams see traceable evidence. Notes are generated with Cortex, grounded only in manifest data, and stored as VARIANT on the release package.

## When to Use
- Drafting notes after scope assembly marks a package READY.
- Regenerating notes after a modify request from the Release Gatekeeper.
- Producing a stakeholder-facing summary or changelog for a release window.
- Highlighting breaking changes, downstream impact, and rollback steps.

## Instructions

1. Store prompt templates in a governed table so wording is versioned and reviewable.

    CREATE TABLE IF NOT EXISTS RELEASE_DB.CORE.NOTE_TEMPLATES (
        AUDIENCE      STRING,      -- BUSINESS | ENGINEERING | COMPLIANCE
        VERSION       INTEGER,
        PROMPT        STRING,
        IS_ACTIVE     BOOLEAN DEFAULT TRUE
    );

    INSERT INTO RELEASE_DB.CORE.NOTE_TEMPLATES VALUES
    ('BUSINESS', 1,
     'You write release notes for business stakeholders. Use plain language, no SQL. Sections: Summary, What is changing, Who is affected, When, Action required. Use only facts in the JSON. If a field is missing, write "Not specified".', TRUE),
    ('ENGINEERING', 1,
     'You write technical release notes. Sections: Scope, Deployment order, Breaking changes, Downstream impact, Rollback plan, Test evidence. Reference CHANGE_ID and object names exactly. Use only facts in the JSON.', TRUE),
    ('COMPLIANCE', 1,
     'You write a change record summary for auditors. Sections: Change list with ticket IDs, Test run IDs, Approvals pending, Data objects with sensitive tags. Use only facts in the JSON. Do not speculate.', TRUE);

2. Build the grounding payload from the package.

    SELECT OBJECT_CONSTRUCT(
               'release_id', RELEASE_ID,
               'target_env', TARGET_ENV,
               'window', PLANNED_WINDOW,
               'manifest', MANIFEST,
               'dependencies', DEPENDENCY_REPORT) AS PAYLOAD
    FROM RELEASE_DB.CORE.RELEASE_PACKAGES
    WHERE RELEASE_ID = :rel_id;

3. Generate one version per audience with structured output.

    CREATE OR REPLACE PROCEDURE RELEASE_DB.CORE.SP_DRAFT_RELEASE_NOTES(REL_ID STRING)
    RETURNS VARIANT
    LANGUAGE SQL
    AS
    $$
    BEGIN
        UPDATE RELEASE_DB.CORE.RELEASE_PACKAGES p
        SET RELEASE_NOTES = n.NOTES, UPDATED_AT = CURRENT_TIMESTAMP()
        FROM (
            SELECT OBJECT_AGG(t.AUDIENCE,
                       AI_COMPLETE(
                           model => 'claude-sonnet-4-5',
                           prompt => t.PROMPT || '\n\nRELEASE_JSON:\n' ||
                                     TO_JSON(OBJECT_CONSTRUCT('release_id', r.RELEASE_ID, 'env', r.TARGET_ENV,
                                         'window', r.PLANNED_WINDOW, 'manifest', r.MANIFEST, 'dependencies', r.DEPENDENCY_REPORT)),
                           model_parameters => {'temperature': 0.1, 'max_tokens': 2000}
                       )::VARIANT) AS NOTES
            FROM RELEASE_DB.CORE.RELEASE_PACKAGES r
            CROSS JOIN RELEASE_DB.CORE.NOTE_TEMPLATES t
            WHERE r.RELEASE_ID = :REL_ID AND t.IS_ACTIVE
        ) n
        WHERE p.RELEASE_ID = :REL_ID;

        RETURN (SELECT RELEASE_NOTES FROM RELEASE_DB.CORE.RELEASE_PACKAGES WHERE RELEASE_ID = :REL_ID);
    END;
    $$;

4. Validate before accepting the notes:
    - Every CHANGE_ID in the manifest appears in the ENGINEERING notes.
    - Every TICKET_ID appears in the COMPLIANCE notes.
    - No object names appear that are not in the manifest or dependency report.
    - Breaking changes (DROP, column removal, type change) appear in BUSINESS notes under Action required.

    SELECT f.value:change_id::STRING AS MISSING_CHANGE
    FROM RELEASE_DB.CORE.RELEASE_PACKAGES p,
         LATERAL FLATTEN(p.MANIFEST:changes) f
    WHERE p.RELEASE_ID = :rel_id
      AND NOT CONTAINS(p.RELEASE_NOTES:ENGINEERING::STRING, f.value:change_id::STRING);

    Regenerate once on failure; if it fails again, keep STATUS = 'DRAFT' and log NOTES_VALIDATION_FAILED.

5. Enrich with history: query release-history-search for prior notes on the same objects so recurring changes use consistent wording.

Best practices:
- Ground strictly in manifest JSON; never let the model invent dates, owners, or impact.
- Keep temperature at or below 0.2 for repeatable output.
- Pin the model name per environment and record it in the audit log.
- Allow a human edit before submission; store edits as a new notes version, never overwrite silently.

Output expectations: RELEASE_NOTES VARIANT with keys BUSINESS, ENGINEERING, COMPLIANCE, each a markdown string, plus a validation result logged to the audit trail.

## Coding Conventions
- Audience keys in uppercase: BUSINESS, ENGINEERING, COMPLIANCE.
- Templates are versioned; increment VERSION rather than editing active rows.
- Prompts end with the JSON payload under the RELEASE_JSON label.
- Notes are markdown with H2 sections matching the template section names.
- Dates in ISO 8601 with time zone.