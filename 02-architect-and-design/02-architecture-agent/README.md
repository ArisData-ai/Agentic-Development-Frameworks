# Architecture Agent

The Architecture Agent recommends Snowflake solution architectures and integration patterns that fit workload, scale, and constraints. It evaluates ingestion, transformation, and consumption options, proposes a reference architecture, and documents the trade-offs for Lead Architect review.

**Segment:** 2 - Architect & Design
**Human owner:** Lead Architect

## Primary Use Case

The agent speeds up architecture decisions by replacing ad hoc debate with documented options, evidence, and trade-offs.

## How It Works

| Stage | Agent responsibility |
| --- | --- |
| Sense | Detects approved requirements and discovery findings that call for an architecture decision |
| Act | Evaluates ingestion, transformation, and consumption options, proposes a reference architecture, and documents trade-offs |
| Hand off | Brings the recommendation and alternatives to the Lead Architect, who selects, adjusts, or rejects the approach |

The agent does not make final architecture decisions, deploy objects, or release designs without Lead Architect approval.

## Skill Files

| Skill | Purpose |
| --- | --- |
| `SKILL` | Defines the end-to-end workflow and intake pattern |
| `architecture-option-evaluation` | Scores options and records trade-offs |
| `architecture-reference-design` | Defines the reference architecture and integration patterns |
| `architecture-knowledge-retrieval` | Retrieves reference patterns and prior decisions with Cortex Search |
| `architecture-decision-handoff` | Packages recommendations and records review outcomes |

Start with `SKILL.md`, which directs the workflow to the other skills.

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── architecture-option-evaluation.md
        └── architecture-reference-design.md
        └── architecture-knowledge-retrieval.md
        └── architecture-decision-handoff.md

## Snowflake Platform Components

| Component | Role |
| --- | --- |
| Cortex Agents | Coordinate the architecture workflow |
| Cortex Search | Retrieves reference patterns and prior design decisions |
| Openflow | Anchors ingestion and integration patterns |
| Streams & Tasks | Detect new work and support incremental processing designs |
| Dynamic Tables | Provide a declarative transformation option |

## Prerequisites

- Snowflake account with access to the required platform features
- Approved requirements and discovery findings
- Curated reference architectures, standards, and prior decisions
- Warehouses for agent, search, and transformation workloads
- Least-privilege roles for agent operators and Lead Architect reviewers

## Core Snowflake Objects

| Object | Purpose |
| --- | --- |
| `ARCH_DB.INTAKE.ARCH_DECISION_REQUEST` | Stores incoming requests |
| `ARCH_DB.INTAKE.ARCH_DECISION_REQUEST_STRM` | Tracks new requests |
| `ARCH_DB.INTAKE.ARCH_INTAKE_T` | Moves approved requests to the work queue |
| `ARCH_DB.INTAKE.ARCH_WORK_QUEUE` | Holds requests ready for processing |
| `ARCH_DB.KNOWLEDGE.ARCH_DOCUMENT_CHUNK` | Stores searchable document chunks |
| `ARCH_DB.KNOWLEDGE.ARCH_PATTERN_SEARCH_SVC` | Searches patterns and prior decisions |
| `ARCH_DB.DECISIONS.ARCH_OPTION_SCORE` | Stores weighted option scores |
| `ARCH_DB.DECISIONS.ARCH_DECISION_LOG` | Records recommendations and review outcomes |

Replace the sample database, schema, warehouse, and object names with the values for your environment.

## Getting Started

1. Create the intake and decision objects defined in the skill files.
2. Load curated architecture content into `ARCH_DOCUMENT_CHUNK`.
3. Create the Cortex Search service and test retrieval.
4. Configure the Cortex Agent with the skills and required tools.
5. Grant the required roles, then resume the intake task:

        ALTER TASK ARCH_DB.INTAKE.ARCH_INTAKE_T RESUME;

6. Submit an approved test request and confirm that the agent creates a `PENDING_REVIEW` decision.
7. Complete a Lead Architect review and confirm that the outcome is recorded in `ARCH_DECISION_LOG`.

## Outputs

- Framed architecture request
- Weighted option comparison
- Reference architecture and alternatives
- Documented trade-offs, assumptions, and risks
- Cited patterns and prior decisions
- Lead Architect review package
- Auditable decision record

## Who Benefits

- **Solution architects:** Compare options with trade-offs already laid out.
- **CTOs and platform owners:** Gain confidence that designs match strategy.
- **Delivery teams:** Build against a clear, agreed pattern.

## Ideal Customer Profile

Enterprises moving to Snowflake with several viable integration paths, where inconsistent architecture decisions have created technical debt.