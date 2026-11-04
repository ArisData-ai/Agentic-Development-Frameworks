---
name: data-modelling-agent
description: This skill defines the end-to-end workflow for the Data Modelling Agent, which turns approved requirements and assessed sources into conceptual, logical, and physical target model drafts, using Cortex Agents for coordination, Cortex Analyst semantic definitions for business alignment, and Dynamic Tables for declarative layers. Use this skill for any task involving orchestrating a data modeling cycle, starting a target model, designing a unified warehouse, consolidating SAP, Oracle, NetSuite, Salesforce, or other source systems, tracking model status, revising a draft, or preparing a model for Lead Architect approval.
---

# Data Modelling Agent

## Overview

The Data Modelling Agent is part of Segment 2: Architect & Design. It designs target data models that reflect business requirements and the assessed state of the sources.

The agent shortens the modeling cycle and keeps designs consistent with business definitions. It drafts and recommends only. The **Lead Architect** owns every approval, change request, and override.

Operating loop:

- **Senses:** Detects approved requirements and assessed sources that need a target model.
- **Acts:** Drafts conceptual, logical, and physical models, proposes keys and relationships, and flags where source data cannot support the design.
- **Hands off:** Submits the model to the Lead Architect.

## When to Use

Use this skill when:

- A new or changed requirement needs a target model.
- Multiple source systems must be consolidated into a unified warehouse.
- Conflicting business definitions are slowing model design.
- A submitted model needs revision after Lead Architect feedback.

Do not use this skill to:

- Select the overall solution architecture or integration pattern. The Architecture Agent owns this.
- Define roles, masking, or row-access policies. The Governance Agent owns this.
- Build production pipelines or commit code. Segment 3 agents act only on approved designs.

## Instructions

1. **Confirm entry conditions.** Start only when the requirements have Lead Validator approval and each in-scope source has an accepted assessment. Set the run to `BLOCKED` and name the missing input if either condition fails.
2. **Register the run.** Create one record for each model version.
3. **Check source fit.** Test whether the assessed sources can support the required entities, grains, and history.
4. **Draft the conceptual and logical models.** Resolve entities, attributes, keys, and relationships. Flag conflicting definitions for a decision.
5. **Draft the physical model.** Translate the approved logical design into Snowflake structures, including declarative Dynamic Table layers where they fit.
6. **Align semantics.** Map model objects to Cortex Analyst semantic definitions and record any gaps.
7. **Submit for review.** Package the model, assumptions, trade-offs, and open issues for the Lead Architect.
8. **Apply the decision.** Revise and resubmit on a change request. Release only an approved version to downstream teams.

Run status values:

    DETECTED -> DRAFTING -> IN_REVIEW -> APPROVED
    DRAFTING -> BLOCKED -> DRAFTING
    IN_REVIEW -> CHANGES_REQUESTED -> DRAFTING

Run tracking table:

    CREATE TABLE IF NOT EXISTS MODEL_DESIGN.CONTROL.MODEL_RUN (
        RUN_ID             STRING        NOT NULL,
        MODEL_NAME         STRING        NOT NULL,
        MODEL_VERSION      STRING        NOT NULL,
        REQUIREMENT_REF    STRING        NOT NULL,
        SOURCE_SCOPE       ARRAY         NOT NULL,
        STATUS             STRING        NOT NULL,
        STATUS_REASON      STRING,
        CREATED_AT         TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP(),
        UPDATED_AT         TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    );

## Coding Conventions

- Use semantic versions for models, such as `1.2.0`.
- Use uppercase Snowflake identifiers with underscores.
- Keep agent-generated artifacts in a design schema until approval.
- Link every model object to a requirement ID and a source reference.
- Label anything inferred as `ASSUMPTION` for reviewer confirmation.
- Never execute DDL in production from this workflow.

## References

- `references/data-model-source-fit.md`: Validates entry inputs and tests source support for the design.
- `references/conceptual-logical-modeling.md`: Defines entities, attributes, keys, relationships, and definition conflict handling.
- `references/snowflake-physical-modeling.md`: Covers physical structures, DDL standards, and Dynamic Table layers.
- `references/semantic-model-alignment.md`: Aligns model objects with business terms and Cortex Analyst semantic definitions.
- `references/data-model-architect-review.md`: Covers the review package, decision capture, overrides, and release.