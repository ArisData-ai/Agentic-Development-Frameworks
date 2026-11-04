# Requirements Agent

**Segment 1: Discover & Assess**

The Requirements Agent turns scattered business inputs into a structured, prioritized requirements baseline before project work begins. It reduces the workshops and manual note consolidation that typically open a data program, while the Lead Validator retains final authority over every requirement.

## How It Works

| Phase | Agent responsibility |
| --- | --- |
| Sense | Detects new stakeholder notes, tickets, meeting transcripts, and policy documents. |
| Act | Extracts functional and data requirements, flags conflicts and gaps, and ranks requirements against stated business objectives. |
| Hand off | Submits the draft baseline to the Lead Validator, who approves, edits, or rejects each requirement before it moves downstream. |

## Skills

| Skill | Coverage |
| --- | --- |
| `SKILL` | Primary skill covering the end-to-end workflow, shared objects, and Cortex Agent setup. |
| `requirements-intake-indexing` | Intake detection, parsing, chunking, and Cortex Search indexing. |
| `requirements-extraction-prioritization` | Requirement extraction, conflict and gap detection, and prioritization. |
| `requirements-validation-handoff` | Lead Validator review, decision logging, baseline versioning, and downstream release. |
| `requirements-governance-lineage` | Access control, sensitive data protection, traceability, and audit evidence. |

## Folder Structure

    requirements-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
       └── requirements-intake-indexing.md
       └── requirements-extraction-prioritization.md
       └── requirements-validation-handoff.md
       └── requirements-governance-lineage.md

## Snowflake Components

- **Cortex Agents:** Orchestrate intake monitoring, extraction, and prioritization.
- **Cortex Search:** Indexes transcripts, tickets, and policy documents for traceable requirements.
- **Snowflake Horizon:** Keeps source documents governed and requirement lineage auditable.
- **Streams & Tasks:** Automate processing when new intake material arrives.

## Prerequisites

- A Snowflake account with access to the required Cortex features.
- A warehouse for intake processing and search indexing.
- Roles for the agent service, Lead Validator, readers, and auditors.
- Defined business objectives for each program.
- Approved access to the source documents and tickets.

## Getting Started

1. Start with `SKILL` to create the shared objects and agent.
2. Configure intake and search with `requirements-intake-indexing`.
3. Generate the draft baseline with `requirements-extraction-prioritization`.
4. Set up human review with `requirements-validation-handoff`.
5. Apply controls and traceability with `requirements-governance-lineage`.

## Outputs

- Structured functional and data requirements
- Conflict and gap findings
- Priority rankings traced to business objectives
- Source citations for each requirement
- Recorded Lead Validator decisions
- A versioned, approved requirements baseline

## Human Accountability

The agent prepares and recommends, but it does not approve requirements. Only requirements approved by the Lead Validator can move downstream to Architect & Design.

## Who Benefits

- **Business analysts:** Spend time on stakeholder conversations, not document reconciliation.
- **Program sponsors:** See objectives traced to requirements from day one.
- **Lead Validators:** Review a clean, conflict-flagged draft instead of raw inputs.

## Ideal Customer Profile

Enterprises launching data modernization programs with many stakeholders, where unclear or shifting requirements cause rework and budget overruns.