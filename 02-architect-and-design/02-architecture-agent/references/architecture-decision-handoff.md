---
name: architecture-decision-handoff
description: This skill defines how to package an architecture recommendation, alternatives, trade-offs, risks, and evidence for Lead Architect review, and records the selected, adjusted, or rejected outcome in an auditable Snowflake decision log. Use this skill for any task involving an architecture decision record, ADR, Lead Architect handoff, approval package, architecture review, trade-off summary, decision log, or downstream release of an approved design.
---

# Architecture Decision Handoff

## Overview

This skill preserves human accountability for architecture decisions. It gives the Lead Architect a complete review package and creates a traceable record for delivery teams.

## When to Use

Use this skill when a recommendation is ready for review, when the Lead Architect responds, or when teams need the approved architecture decision. Do not use it to bypass review or infer approval from silence.

## Instructions

1. Confirm that the package includes the problem, recommendation, alternatives, trade-offs, evidence, assumptions, risks, and open questions.
2. Include at least one viable alternative, unless a recorded hard constraint removes every other option.
3. Submit the decision with the status `PENDING_REVIEW`.
4. Accept only these Lead Architect outcomes:
    - `SELECTED`: The recommendation proceeds unchanged.
    - `ADJUSTED`: The approach proceeds with recorded changes.
    - `REJECTED`: The approach does not proceed.
5. Record the reviewer, timestamp, and rationale.
6. Release only `SELECTED` or `ADJUSTED` decisions to downstream teams.
7. For a rejected decision, record the reason and start a new evaluation only when requested.
8. Create a new decision version for any material change after review.

Use this decision log:

    CREATE TABLE IF NOT EXISTS ARCH_DB.DECISIONS.ARCH_DECISION_LOG (
        DECISION_ID      STRING NOT NULL,
        DECISION_VERSION NUMBER NOT NULL,
        REQUEST_ID       STRING NOT NULL,
        RECOMMENDATION   VARIANT NOT NULL,
        ALTERNATIVES     VARIANT NOT NULL,
        STATUS           STRING NOT NULL,
        REVIEWED_BY      STRING,
        REVIEWED_AT      TIMESTAMP_NTZ,
        REVIEW_RATIONALE STRING,
        CREATED_AT       TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    );

Use this review package template:

    Decision ID:
    Request ID:
    Decision required:
    Business objective:
    Recommended approach:
    Alternatives considered:
    Trade-offs:
    Evidence and citations:
    Assumptions:
    Risks and mitigations:
    Cost and operational impact:
    Open questions:
    Downstream dependencies:
    Requested outcome: SELECT, ADJUST, or REJECT

## Coding Conventions

- Keep decision records append-only for each version.
- Use stable decision IDs and increment `DECISION_VERSION` for revisions.
- Restrict review outcomes to authorized Lead Architect roles.
- Store structured recommendation content in `VARIANT` and readable rationale in text fields.
- Link every decision to its request, evaluation version, and cited sources.s