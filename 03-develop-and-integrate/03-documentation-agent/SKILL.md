---
name: documentation-agent-core
description: This skill defines the end-to-end operating model for a Documentation Agent that converts new or changed pipelines, models, and code into reviewed technical documentation, coordinating change detection, draft generation, Engineering Team approval, and publication through Cortex Agents, Cortex Search, and Snowflake Horizon. Use this skill for any task involving documentation agent design, deployment, orchestration, role setup, workflow planning, documentation state tracking, or troubleshooting across the sense, act, and hand-off stages, including requests to automate data dictionaries, keep technical documentation current, reduce documentation drift, or support onboarding, support, and audit readiness.
---

# Documentation Agent Core

## Overview

The Documentation Agent keeps technical documentation current by producing it as delivery work happens. It operates in Segment 3 (Develop & Integrate) and follows three stages:

- **Sense:** Detect new or changed pipelines, models, and code.
- **Act:** Generate data dictionaries, lineage summaries, and technical descriptions, and flag undocumented objects.
- **Hand off:** Submit drafts to the Engineering Team, who correct and approve them before publication.

The agent drafts documentation but never owns the final decision. The Engineering Team remains accountable for all published content.

## When to Use

- Designing or deploying the Documentation Agent in a Snowflake account.
- Defining the workflow, roles, or shared objects across agent stages.
- Tracing an item through detection, drafting, review, and publication.
- Planning automation for environments with fast platform change, high team turnover, or missing documentation.

Use the sibling skills listed under References for stage-specific implementation.

## Instructions

1. Create a dedicated database, schema, warehouse, and role set.

        CREATE DATABASE IF NOT EXISTS DOC_AGENT_DB;
        CREATE SCHEMA IF NOT EXISTS DOC_AGENT_DB.CORE;
        CREATE WAREHOUSE IF NOT EXISTS DOC_AGENT_WH
            WAREHOUSE_SIZE = 'XSMALL'
            AUTO_SUSPEND = 60
            AUTO_RESUME = TRUE;

        CREATE ROLE IF NOT EXISTS DOC_AGENT_RUNNER;
        CREATE ROLE IF NOT EXISTS DOC_REVIEWER;
        CREATE ROLE IF NOT EXISTS DOC_READER;

2. Assign responsibilities through a clear component map.

    | Component | Responsibility |
    | --- | --- |
    | Streams and Tasks | Queue processing and scheduled execution |
    | Cortex Agents | Trigger and coordinate documentation runs |
    | Snowflake Horizon | Supply lineage, classification, and metadata |
    | Cortex Search | Make approved documentation searchable |
    | Engineering Team | Correct, approve, or reject drafts |

3. Track every item through one lifecycle.

        DETECTED -> DRAFTED -> IN_REVIEW -> APPROVED -> PUBLISHED
                                   |
                                   +-> CHANGES_REQUESTED -> DRAFTED
                                   +-> REJECTED

4. Enforce these guardrails.

    - Publish only content approved by the Engineering Team.
    - Base generated statements on available metadata, definitions, or code.
    - Flag unknown business meaning for review; do not invent it.
    - Exclude row-level data and secrets from prompts and drafts.
    - Keep the review role separate from the agent execution role.

5. Verify that each run produces these outputs.

    - A data dictionary, lineage summary, and technical description, as applicable.
    - A list of undocumented objects and unresolved questions.
    - A review record showing the approver, decision, and timestamp.
    - A searchable published version with a link to its source object.

6. Track operating measures: documentation coverage, detection-to-draft time, review turnaround, rejection rate, and undocumented object count.

## Coding Conventions

- Use uppercase for SQL keywords and Snowflake identifiers.
- Use fully qualified object names in all scripts.
- Prefix agent objects with `DOC_`.
- Replace `DOC_AGENT_DB` and related names through deployment configuration.
- Make setup scripts idempotent with `IF NOT EXISTS` where supported.
- Keep environment-specific values outside reusable logic.

## References

| Skill file | Coverage |
| --- | --- |
| `references/documentation-change-detection.md` | Change capture, queueing, task scheduling, and undocumented object detection |
| `references/documentation-content-generation.md` | Metadata collection, data dictionaries, lineage summaries, technical descriptions, and draft quality checks |
| `references/documentation-review-publication.md` | Engineering Team review, approval records, publication, Cortex Search setup, and version control |