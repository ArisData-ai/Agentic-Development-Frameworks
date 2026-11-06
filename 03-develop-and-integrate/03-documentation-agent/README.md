# Documentation Agent

*Segment 3: Develop & Integrate*

The Documentation Agent keeps technical documentation current by writing it as delivery work happens. It detects new or changed pipelines, models, and code, generates draft documentation, and routes each draft to the Engineering Team for correction and approval before publication.

## Why It Matters

Documentation often falls behind delivery, which slows onboarding and support and makes audits harder. This agent makes documentation part of the delivery workflow while keeping the Engineering Team accountable for what gets published.

## How It Works

| Stage | Agent or team action |
| --- | --- |
| **Sense** | Detects new or changed pipelines, models, and code |
| **Act** | Generates data dictionaries, lineage summaries, and technical descriptions, and flags undocumented objects |
| **Hand off** | Submits drafts to the Engineering Team for correction and approval |

Documentation moves through the following lifecycle:

    DETECTED -> DRAFTED -> IN_REVIEW -> APPROVED -> PUBLISHED
                               |
                               +-> CHANGES_REQUESTED -> DRAFTED
                               +-> REJECTED

## Skill Files

| File | Use it for |
| --- | --- |
| `SKILL.md` | Operating model, setup, roles, lifecycle, guardrails, and outputs |
| `documentation-change-detection.md` | Change capture, queueing, scheduling, and undocumented object reporting |
| `documentation-content-generation.md` | Metadata collection, draft generation, output structure, and quality checks |
| `documentation-review-publication.md` | Review decisions, audit records, versioned publication, and Cortex Search |

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── documentation-change-detection.md
        └── documentation-content-generation.md
        └── documentation-review-publication.md

## Snowflake Components

| Component | Role |
| --- | --- |
| **Cortex Agents** | Trigger and coordinate documentation work |
| **Cortex Search** | Makes published documentation searchable |
| **Snowflake Horizon** | Supplies lineage and metadata |
| **Streams and Tasks** | Support queue processing and scheduled detection |

## Prerequisites

- A Snowflake account with access to the required Cortex features
- A dedicated database, schema, warehouse, and role set
- Access to `SNOWFLAKE.ACCOUNT_USAGE` and the target object metadata
- Engineering Team reviewers assigned to the `DOC_REVIEWER` role
- A CI/CD or Git process that can send code change events to the queue (if code changes are in scope)

## Getting Started

1. Follow `documentation-agent-core.md` to create the shared database, schema, warehouse, and roles.
2. Create the change queue using `documentation-change-detection.md`.
3. Create the draft table using `documentation-content-generation.md`.
4. Create the review and publication tables using `documentation-review-publication.md`.
5. Implement the `RUN_DOC_GENERATION` procedure for your environment. The procedure must consume the queue stream and write validated drafts to `DOC_DRAFTS`.
6. Create the detection and generation tasks, then resume them.
7. Create the Cortex Search service and grant access to documentation consumers.
8. Test the full workflow on a non-production object before expanding its scope.

## Outputs

- Data dictionaries
- Lineage summaries
- Technical descriptions
- Undocumented object reports
- Review and approval records
- Versioned, searchable published documentation

## Guardrails

- Publish only content approved by the Engineering Team.
- Generate documentation from available metadata, definitions, and code.
- Flag unknown business meaning for review.
- Exclude row-level data, secrets, and credentials from prompts and drafts.
- Keep the agent execution role separate from the review role.

## Who Benefits

- **New team members:** Onboard using accurate documentation.
- **Support teams:** Find answers without chasing authors.
- **Auditors:** Trace objects to documented descriptions.

## Ideal Customer Profile

Organizations with fast-changing data platforms and high team turnover, where missing documentation slows onboarding and support.