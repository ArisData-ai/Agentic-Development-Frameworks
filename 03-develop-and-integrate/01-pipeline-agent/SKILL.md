---
name: pipeline-agent-orchestration
description: This skill defines the end-to-end Pipeline Agent workflow that converts approved designs and source mappings into reviewable ELT pipeline definitions, coordinating Cortex Agents, Openflow, Dynamic Tables, and Streams and Tasks across intake, build, and Engineering Team handoff. Use this skill for any task involving building, migrating, or maintaining ELT pipelines end to end, including planning a pipeline build, clearing a pipeline backlog, moving SAP, Oracle, or Workday data into Snowflake, coordinating Cortex Agents, or deciding which pipeline skill applies.
---

# Pipeline Agent Orchestration

## Overview
The Pipeline Agent operates in Segment 3: Develop & Integrate. It builds and maintains ELT pipelines from approved designs so that ingestion is ready sooner and stays consistent.

The agent follows a three-stage operating model:

- **Sense:** Detect approved designs and source mappings that are ready for build.
- **Act:** Assemble ingestion and transformation flows, set dependencies and schedules, and flag mapping gaps it cannot resolve.
- **Hand off:** Submit pipeline definitions to the Engineering Team for review, adjustment, and approval before merge.

The Engineering Team retains decision authority. The agent never merges its own work or deploys it to production.

## When to Use
- Build a new ELT pipeline from an approved design.
- Migrate an existing pipeline to Snowflake from an approved mapping.
- Update a pipeline after an approved design or mapping change.
- Plan and sequence a backlog of pipeline builds.

Do not use this workflow for unapproved designs. Return them to the Lead Architect through the design process.

## Instructions
1. **Validate the build input.** Confirm that the design and source mappings are approved and complete. Follow `pipeline-design-intake`.
2. **Create the build plan.** Use Cortex Agents to coordinate the required ingestion, transformation, and orchestration tasks.
3. **Assemble ingestion.** Build or configure source flows in Openflow. Follow `pipeline-openflow-ingestion`.
4. **Assemble transformations.** Build incremental layers and set dependencies and schedules. Follow `pipeline-transformation-orchestration`.
5. **Record unresolved items.** Flag mapping gaps instead of inventing logic. Continue only with work that the gap does not affect.
6. **Submit for review.** Package the definitions and evidence for the Engineering Team. Follow `pipeline-engineering-handoff`.
7. **Apply review outcomes.** Revise the pipeline as directed and resubmit it. Treat approval as specific to the reviewed version.

Coordinate with adjacent agents at these boundaries:

- **Code Generation Agent:** Owns SQL generation and legacy code conversion.
- **Documentation Agent:** Owns data dictionaries, lineage summaries, and technical descriptions.
- **Segment 4 agents:** Own formal data quality, testing, and reconciliation after build.

## Coding Conventions
- Trace every pipeline object to an approved design and mapping version.
- Store pipeline definitions in version control.
- Build and validate in non-production environments only.
- Use environment parameters instead of hard-coded databases, warehouses, or credentials.
- Keep definitions idempotent so that they can be safely redeployed.
- Use this standard build status vocabulary:

        READY_FOR_BUILD
        BLOCKED_MAPPING_GAP
        IN_BUILD
        READY_FOR_REVIEW
        CHANGES_REQUESTED
        APPROVED_FOR_MERGE

## References
| Skill file | Use it for |
|---|---|
| `references/pipeline-design-intake.md` | Checking build readiness, validating mappings, and flagging gaps |
| `references/pipeline-openflow-ingestion.md` | Configuring source connectivity and landing raw data through Openflow |
| `references/pipeline-transformation-orchestration.md` | Building Dynamic Tables, Streams and Tasks, dependencies, and schedules |
| `references/pipeline-engineering-handoff.md` | Packaging pipeline definitions and managing Engineering Team review |