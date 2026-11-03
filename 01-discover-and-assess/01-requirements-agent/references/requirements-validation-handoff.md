---
name: requirements-validation-handoff
description: This skill defines how the Requirements Agent packages a draft baseline for the Lead Validator, records approve, edit, and reject decisions, and publishes versioned approved requirements through Snowflake tables, views, and Streams and Tasks. Use this skill for any task involving human-in-the-loop review, Lead Validator workflows, requirement approval, baseline sign-off, reviewer queues, version control, change requests, or downstream handoff to Architect & Design agents.
---

# Requirements Validation and Handoff

## Overview

This skill covers the Hand-off phase. It gives the Lead Validator a clean, conflict-flagged draft and ensures that no requirement moves downstream without a recorded human decision.

## When to Use

- Build or update the Lead Validator review queue.
- Record approve, edit, or reject decisions.
- Publish a new baseline version.
- Reopen requirements when intake material changes.

## Instructions

1. Create an immutable decision log.

        CREATE TABLE IF NOT EXISTS VALIDATION_DECISION (
            DECISION_ID STRING DEFAULT UUID_STRING(),
            REQUIREMENT_ID STRING NOT NULL,
            REQUIREMENT_VERSION NUMBER NOT NULL,
            DECISION STRING NOT NULL,
            ORIGINAL_STATEMENT STRING NOT NULL,
            FINAL_STATEMENT STRING,
            REASON STRING,
            DECIDED_BY STRING DEFAULT CURRENT_USER(),
            DECIDED_AT TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Present each requirement with its rank, rationale, source citations, confidence, and open findings.
3. List contradictions and high-priority items first.
4. Require one decision for each requirement:

   - `APPROVE`: Accept the statement unchanged.
   - `EDIT`: Save the validated wording, increment the version, and approve it.
   - `REJECT`: Exclude the requirement and record the reason.

5. Block approval while a blocking finding remains open.
6. Publish only approved requirements through a stable view.

        CREATE OR REPLACE VIEW APPROVED_REQUIREMENTS_V AS
        SELECT
            PROGRAM_ID, REQUIREMENT_ID, VERSION,
            REQUIREMENT_TYPE, STATEMENT,
            ACCEPTANCE_CRITERIA, OBJECTIVE_ID,
            PRIORITY_SCORE, PRIORITY_RANK
        FROM REQUIREMENT
        WHERE STATUS = 'APPROVED';

7. Create a numbered baseline snapshot when the Lead Validator signs off.
8. When new intake affects an approved requirement, create a new draft version and return it for review. Do not change the approved version in place.
9. Notify downstream teams with the baseline version and a summary of changes.

## Coding Conventions

- Grant write access for decisions only to the Lead Validator role.
- Prevent service roles from setting `APPROVED` or `REJECTED`.
- Treat decision records as append-only.
- Require a reason for edits and rejections.
- Reference a baseline version in all downstream work.