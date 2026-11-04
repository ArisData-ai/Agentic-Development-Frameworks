# Data Modelling Agent

*Segment 2: Architect & Design*

The Data Modelling Agent designs target data models that reflect business requirements and the assessed state of the sources. It shortens the modeling cycle while keeping designs consistent with business definitions.

The agent drafts and recommends. The **Lead Architect** approves each model, requests changes, or overrides modeling choices.

## How It Works

| Stage | What happens |
| --- | --- |
| **Senses** | Detects approved requirements and assessed sources that need a target model. |
| **Acts** | Drafts conceptual, logical, and physical models, proposes keys and relationships, and flags where source data cannot support the design. |
| **Hands off** | Submits the model to the Lead Architect for a decision. |

## Skill Files

| Skill | Purpose |
| --- | --- |
| `SKILL` | Primary workflow, entry conditions, run tracking, and skill routing |
| `data-model-source-fit` | Input validation and source support testing |
| `conceptual-logical-modeling` | Entities, attributes, grain, keys, relationships, and definition conflicts |
| `snowflake-physical-modeling` | Snowflake structures, DDL standards, and Dynamic Table layers |
| `semantic-model-alignment` | Business term mapping and Cortex Analyst semantic definitions |
| `data-model-architect-review` | Review package, decision capture, versioning, and release |

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── data-model-source-fit.md
        └── conceptual-logical-modeling.md
        └── snowflake-physical-modeling.md
        └── semantic-model-alignment.md
        └── data-model-architect-review.md

## Workflow

1. Confirm that requirements and source assessments are approved.
2. Test whether the sources support the planned design.
3. Draft the conceptual and logical models.
4. Translate the design into a Snowflake physical model.
5. Align the model with business terms and semantic definitions.
6. Submit the review package to the Lead Architect.
7. Revise the model if changes are requested.
8. Release the approved version to Segment 3 teams.

## Inputs and Outputs

**Inputs**

- Requirements approved by the Lead Validator
- Accepted source assessments and discovery findings
- Approved business definitions and reporting needs
- Project standards and configuration values

**Outputs**

- Conceptual, logical, and physical model specifications
- Proposed keys and relationships
- Snowflake DDL and Dynamic Table definitions
- Source-fit findings and design gaps
- Semantic alignment report
- Lead Architect review package and decision log

## Snowflake Platform Relevance

- **Cortex Agents:** Coordinate model generation and revision.
- **Cortex Analyst:** Uses semantic definitions to keep models aligned with business terms.
- **Dynamic Tables:** Support declarative layers in the physical design.

## Agent Boundaries

| Area | Owner |
| --- | --- |
| Solution architecture and integration patterns | Architecture Agent |
| Roles, masking, and row-access policies | Governance Agent |
| Pipeline build | Pipeline Agent |
| Code generation | Code Generation Agent |
| Final model decision | Lead Architect |

## Guardrails

- Start only with approved requirements and accepted source assessments.
- Do not execute generated DDL in production.
- Do not resolve conflicting business definitions silently.
- Label inferred details as assumptions for review.
- Do not treat silence or partial feedback as approval.
- Release only approved model versions to downstream teams.

## Who Benefits

- **Data modelers:** Start from a reasoned draft instead of a blank diagram.
- **Analytics leaders:** Get models aligned to reporting needs.
- **Downstream engineers:** Receive clear, consistent structures to build against.

## Ideal Customer Profile

Organizations consolidating multiple source systems into a unified warehouse, where conflicting definitions slow model design.