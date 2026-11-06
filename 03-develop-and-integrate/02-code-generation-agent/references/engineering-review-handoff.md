---
name: engineering-review-handoff
description: This skill defines how to package generated Snowflake code, traceability records, confidence ratings, and open flags for Engineering Team review, and how to process edits, approvals, rejections, and resubmissions without committing code. Use this skill for any task involving preparing a pull request summary, creating a review manifest, listing translation flags, handing off draft SQL or Snowpark code, applying reviewer comments, tracking approval status, or confirming readiness for commit.
---

# Engineering Review Handoff

## Overview

This skill keeps the Engineering Team accountable for all generated code. It gives reviewers the context needed to review, edit, approve, or reject code before commit.

## When to Use

- Generated or converted code is ready for review.
- A reviewer returns comments or edits.
- A team needs a pull request summary or review manifest.
- A team needs to confirm whether code is ready for commit.

## Instructions

1. Confirm that each code unit has a header, source reference, and draft revision.
2. Group code by specification, legacy object, or logical change set.
3. List all confidence ratings, flags, assumptions, and standards applied.
4. Put BLOCKER items first, followed by REVIEW items.
5. Submit the package to the Engineering Team.
6. Record each reviewer decision as `APPROVED`, `CHANGES_REQUESTED`, or `REJECTED`.
7. For `CHANGES_REQUESTED`, update only the affected code and increment the draft revision.
8. For `REJECTED`, record the reason and stop work on that unit until a new approved request exists.
9. Mark code as ready for commit only after an authorized reviewer approves it and all BLOCKER flags are resolved.
10. Leave the commit action to the Engineering Team or its approved source control process.

Readiness requires:

- Approved source specification or conversion request
- Complete source-to-code traceability
- No unresolved BLOCKER flags
- Reviewer decisions on all REVIEW flags
- Recorded reviewer identity, date, and draft revision
- No hardcoded secrets or environment-specific credentials

## Coding Conventions

Use this review manifest:

    change_set_id: <ID>
    source_reference: <specification or conversion request ID>
    source_version: <version>
    draft_revision: <number>
    standards_applied:
        - <document ID and version>
    code_units:
        - file: <file name>
          target_object: <fully qualified object name>
          source: <specification or legacy object>
          confidence: <HIGH, MEDIUM, or LOW>
          status: <PENDING_REVIEW, APPROVED, CHANGES_REQUESTED, or REJECTED>
    open_flags:
        - flag_id: <FLAG_ID>
          severity: <BLOCKER or REVIEW>
          file: <file name>
          line: <line number>
          decision_needed: <specific question>
    assumptions:
        - <assumption requiring confirmation>
    reviewer:
        name: <reviewer name>
        decision: <decision>
        decision_date: <YYYY-MM-DD>

Use this pull request summary:

    Summary: <what the change generates or converts>
    Source: <specification or legacy object and version>
    Target objects: <fully qualified objects>
    Confidence: <HIGH count>, <MEDIUM count>, <LOW count>
    Blockers: <count and flag IDs>
    Review items: <count and flag IDs>
    Standards applied: <document IDs and versions>
    Status: DRAFT, pending Engineering Team approval