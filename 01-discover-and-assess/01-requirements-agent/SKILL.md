---
name: requirements-agent-workflow
description: This skill defines the end-to-end Requirements Agent workflow that converts stakeholder notes, tickets, meeting transcripts, and policy documents into a structured, prioritized, and validated requirements baseline through Cortex Agents, Cortex Search, and Snowflake Horizon. Use this skill for any task involving building, configuring, running, or troubleshooting the Requirements Agent, orchestrating intake through Lead Validator review, creating a requirements baseline, or starting a data modernization or migration program.
---

# Requirements Agent Workflow

## Overview

The Requirements Agent turns scattered business inputs into a prioritized requirements baseline before project work begins. It belongs to Segment 1, Discover & Assess. The agent follows a sense, act, and hand-off pattern, and the Lead Validator retains final authority over every requirement.

## When to Use

- Deploy or configure the Requirements Agent for a new program.
- Run the full workflow from intake to baseline.
- Determine which phase owns a task or failure.
- Prepare an approved baseline for downstream Architect & Design agents.

## Instructions

1. Create the shared objects in `REQ_DB.REQ_CORE`.

        CREATE TABLE IF NOT EXISTS BUSINESS_OBJECTIVE (
            OBJECTIVE_ID STRING PRIMARY KEY,
            PROGRAM_ID STRING NOT NULL,
            OBJECTIVE_TEXT STRING NOT NULL,
            OBJECTIVE_WEIGHT NUMBER(3,2) NOT NULL
        );

        CREATE TABLE IF NOT EXISTS REQUIREMENT (
            REQUIREMENT_ID STRING PRIMARY KEY,
            PROGRAM_ID STRING NOT NULL,
            REQUIREMENT_TYPE STRING NOT NULL,
            STATEMENT STRING NOT NULL,
            ACCEPTANCE_CRITERIA STRING,
            OBJECTIVE_ID STRING,
            PRIORITY_SCORE NUMBER(5,2),
            PRIORITY_RANK NUMBER,
            STATUS STRING DEFAULT 'DRAFT',
            VERSION NUMBER DEFAULT 1,
            UPDATED_AT TIMESTAMP_TZ DEFAULT CURRENT_TIMESTAMP()
        );

2. Run the phases in order:

   - **Sense:** Detect, parse, and index new intake material.
   - **Act:** Extract requirements, flag conflicts and gaps, and rank the results.
   - **Hand off:** Submit the draft baseline to the Lead Validator.
   - **Govern:** Apply access controls and preserve lineage throughout.

3. Register the Cortex Agent with intake search as a tool.

        CREATE OR REPLACE AGENT REQ_DB.REQ_CORE.REQUIREMENTS_AGENT
            FROM SPECIFICATION
            $$
            models:
              orchestration: auto
            instructions:
              orchestration: >
                Extract requirements only from indexed intake material.
                Cite a source for every requirement. Flag conflicts and
                gaps; never resolve them silently. Never approve a
                requirement.
            tools:
              - tool_spec:
                  type: cortex_search
                  name: intake_search
                  description: Searches program intake material.
            tool_resources:
              intake_search:
                name: REQ_DB.REQ_CORE.INTAKE_SEARCH
                max_results: 10
            $$;

4. Enforce the requirement lifecycle: `DRAFT` to `IN_REVIEW`, then `APPROVED` or `REJECTED`.
5. Release only `APPROVED` requirements to downstream agents.

## Coding Conventions

- Use uppercase Snowflake identifiers and `REQ_` prefixes for program-level objects.
- Format requirement IDs as `<PROGRAM_ID>-REQ-<0001>`.
- Scope every query and write by `PROGRAM_ID`.
- Keep agent instructions in version control.
- Use idempotent DDL and `MERGE` statements so reruns do not create duplicates.

## References

- `references/requirements-intake-indexing.md`: Intake detection, parsing, chunking, and Cortex Search indexing.
- `references/requirements-extraction-prioritization.md`: Requirement extraction, conflict and gap detection, and objective-based ranking.
- `references/requirements-validation-handoff.md`: Lead Validator review, decisions, baseline versioning, and downstream release.
- `references/requirements-governance-lineage.md`: Source document governance, access control, traceability, and audit evidence.