# Pipeline Agent

*Segment 3: Develop & Integrate*

The Pipeline Agent builds and maintains ELT pipelines from approved designs so ingestion is ready sooner and stays consistent. It assembles pipeline definitions for Engineering Team review; it does not merge changes or deploy them to production.

## What It Does

| Stage | Responsibility |
|---|---|
| **Sense** | Detects approved designs and source mappings ready for build |
| **Act** | Assembles ingestion and transformation flows, sets dependencies and schedules, and flags mapping gaps it cannot resolve |
| **Hand off** | Submits pipeline definitions to the Engineering Team for review, adjustment, and approval before merge |

## Primary Use Case

Accelerates delivery of ELT pipelines without sacrificing standards.

## Who Benefits

- **Data engineers:** Spend time on hard logic instead of repetitive plumbing.
- **Delivery managers:** Hit timelines with fewer rebuilds.
- **Business teams:** Get data sooner.

## Snowflake Platform Relevance

| Component | Role |
|---|---|
| **Openflow** | Handles ingestion from source systems |
| **Dynamic Tables** | Run declarative incremental transformations |
| **Streams & Tasks** | Run explicit incremental processing and orchestration |
| **Cortex Agents** | Coordinate pipeline assembly |

## Ideal Customer Profile

Organizations with a large backlog of pipelines to build or migrate, where engineering capacity limits delivery speed.

## Skill Files

Start with `SKILL.md`, then use the skill that matches the current task.

| Skill | Purpose |
|---|---|
| `SKILL` | Coordinates the end-to-end workflow and directs work to the appropriate skill |
| `pipeline-design-intake` | Validates approved inputs, creates the build manifest, and records mapping gaps |
| `pipeline-openflow-ingestion` | Configures source connectivity and raw data landing through Openflow |
| `pipeline-transformation-orchestration` | Builds incremental transformations, dependencies, and schedules |
| `pipeline-engineering-handoff` | Packages pipeline definitions and manages Engineering Team review |

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── pipeline-design-intake.md
        └── pipeline-openflow-ingestion.md
        └── pipeline-transformation-orchestration.md
        └── pipeline-engineering-handoff.md

## Workflow

    Approved design and source mappings
        -> Validate inputs and create the build manifest
        -> Assemble Openflow ingestion
        -> Assemble transformations, dependencies, and schedules
        -> Validate in non-production
        -> Submit to the Engineering Team
        -> Revise as directed or approve for merge
        -> Proceed to Segment 4: Test & Validate

## Human Accountability

The **Engineering Team** owns the final review. It can adjust, approve, or reject each pipeline definition before merge. Any change made after approval requires a new review.

## Getting Started

1. Provide the approved design and source mappings.
2. Ask the agent to validate build readiness and create the build manifest.
3. Resolve any blocking mapping gaps.
4. Ask the agent to build the ingestion and transformation definitions.
5. Review the submitted package and record the Engineering Team decision.

Example prompt:

    Validate the approved sales orders design and source mapping. If it is ready for build, create the pipeline definitions and prepare the Engineering Team review package. Flag any mapping gaps instead of making assumptions.