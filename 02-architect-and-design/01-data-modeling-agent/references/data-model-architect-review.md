---
name: data-model-architect-review
description: This skill defines how the Data Modelling Agent packages model drafts for Lead Architect review, captures approvals, change requests, and overrides, manages revisions, and releases approved versions with a full decision record. Use this skill for any task involving submitting a data model, writing a review summary, listing assumptions and trade-offs, applying architect feedback, logging an override, versioning a model, comparing model versions, or handing an approved design to pipeline and code generation teams.
---

# Data Model Architect Review

## Overview

This skill enforces human accountability for the design. The agent prepares evidence and recommendations, while the Lead Architect decides. No model reaches downstream teams without an approved version.

## When to Use

Use this skill when:

- A model draft is ready for submission.
- The Lead Architect approves, requests changes, or overrides a choice.
- A model needs a new version or a change summary.
- An approved design is ready for handoff.

## Instructions

### Submit

1. Confirm that the model has traceability to requirements and sources.
2. Assemble the review package.
3. Set the run status to `IN_REVIEW`.

Review package contents:

    1. Model summary and version
    2. Conceptual, logical, and physical specifications
    3. Requirement traceability matrix
    4. Source-fit findings and open gaps
    5. Definition conflicts and recommendations
    6. Semantic alignment report
    7. Key decisions, alternatives, and trade-offs
    8. Assumptions needing confirmation
    9. Changes since the previous version

### Capture the Decision

1. Record each decision, reviewer, reason, and affected objects.
2. On `APPROVED`, lock that version.
3. On `CHANGES_REQUESTED`, return the run to `DRAFTING` and address each request.
4. On `OVERRIDDEN`, apply the Lead Architect's choice as stated.
5. Flag any downstream conflict that an override creates, but do not reverse the decision.

Decision log table:

    CREATE TABLE IF NOT EXISTS MODEL_DESIGN.CONTROL.MODEL_DECISION (
        DECISION_ID       STRING        NOT NULL,
        RUN_ID            STRING        NOT NULL,
        MODEL_VERSION     STRING        NOT NULL,
        OBJECT_REF        STRING,
        DECISION_TYPE     STRING        NOT NULL,
        DECISION_REASON   STRING,
        DECIDED_BY        STRING        NOT NULL,
        DECIDED_AT        TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    );

Decision types:

    APPROVED
    CHANGES_REQUESTED
    OVERRIDDEN

### Release

1. Confirm that the final version has no unresolved blocker.
2. Publish the approved specification and DDL to the agreed design location.
3. Notify the Pipeline, Code Generation, and Documentation Agents of the approved version.

## Coding Conventions

- Treat approved versions as immutable.
- Use a new version for any change after approval.
- Separate facts, assumptions, and recommendations in every package.
- Reference the exact model version in every decision.
- Never infer approval from silence or partial feedback.
- Keep each override in the decision log, even after later revisions.