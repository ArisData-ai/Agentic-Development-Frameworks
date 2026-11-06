---
name: pipeline-engineering-handoff
description: This skill defines how the Pipeline Agent packages pipeline definitions, validation evidence, open gaps, and traceability records for Engineering Team review, then processes approval, adjustment, or rejection before merge. Use this skill for any task involving pipeline pull requests, merge readiness, engineering review, submitting ELT definitions, responding to review comments, resubmitting changes, creating review checklists, or preparing a pipeline for Segment 4 testing.
---

# Pipeline Engineering Handoff

## Overview
This skill controls the Hand off stage of the Pipeline Agent. It gives the Engineering Team the evidence needed to review, adjust, and approve a pipeline before merge. The agent can prepare and revise a submission, but only the Engineering Team can approve it.

## When to Use
- A pipeline build is ready for Engineering Team review.
- Reviewers request a change.
- A team needs to confirm merge readiness.
- An approved pipeline is ready to proceed to Segment 4: Test & Validate.

## Instructions
1. **Assemble the review package.** Include:
    - Pipeline purpose and scope
    - Design and mapping versions
    - Changed Openflow, Dynamic Table, stream, and task definitions
    - Dependencies and schedules
    - Non-production validation evidence
    - Open gaps, risks, and any deviations from the design
    - Rollback or recovery notes
2. **Confirm submission readiness.** Do not submit work with undisclosed failures, unresolved blocking gaps, or missing traceability.
3. **Submit for review.** Set status to `READY_FOR_REVIEW` and assign the responsible Engineering Team reviewers.
4. **Process the outcome.**
    - **Approved:** Set status to `APPROVED_FOR_MERGE` and record the reviewer, date, and approved version.
    - **Adjusted:** Set status to `CHANGES_REQUESTED`, apply the requested changes, revalidate, and resubmit.
    - **Rejected:** Stop the merge path, record the reason, and return the issue to its owner.
5. **Protect the approval.** Request a new review if any reviewed definition changes after approval.
6. **Prepare the downstream handoff.** After merge, provide the approved definitions and pipeline context needed for Segment 4 validation.

## Coding Conventions
- Use one pull request for one logical pipeline change.
- Reference gap IDs and manifest versions in the submission.
- Report validation results as observed; do not imply success for checks that were not run.
- Use this pull request template:

        ## Pipeline Summary
        Pipeline:
        Design version:
        Mapping version:
        Build status: READY_FOR_REVIEW

        ## Changes
        - Ingestion:
        - Transformations:
        - Dependencies and schedules:

        ## Validation Evidence
        - Connectivity:
        - Object creation:
        - Incremental behavior:
        - Rerun behavior:

        ## Open Items
        - Gaps:
        - Risks:
        - Design deviations:

        ## Recovery
        - Rollback or recovery steps:

        ## Engineering Team Decision
        - Reviewer:
        - Decision:
        - Date: