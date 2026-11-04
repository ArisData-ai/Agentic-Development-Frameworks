---
name: conceptual-logical-modeling
description: This skill defines how the Data Modelling Agent converts approved requirements and source-fit findings into conceptual and logical models, including entities, attributes, grain, keys, relationships, history rules, and a documented record of conflicting business definitions. Use this skill for any task involving drafting an ERD, identifying entities, proposing primary or foreign keys, choosing surrogate keys, defining fact and dimension grain, applying dimensional or normalized patterns, handling slowly changing dimensions, or reconciling customer, product, and account definitions across source systems.
---

# Conceptual and Logical Modeling

## Overview

This skill produces a reasoned design draft so modelers do not start from a blank diagram. It keeps business meaning separate from physical implementation and makes every key design choice reviewable.

## When to Use

Use this skill when:

- Creating a new conceptual or logical model.
- Extending a model for a new requirement or source.
- Proposing keys, grain, relationships, or history handling.
- Reconciling conflicting definitions from multiple source systems.

## Instructions

### Conceptual Model

1. Identify the core business entities from the approved requirements.
2. Define each entity in business language.
3. Record the relationships and business rules between entities.
4. Trace every entity to at least one requirement.

### Logical Model

1. Choose the modeling pattern that fits the requirement, and record why.
2. State the grain of each entity or fact in one sentence.
3. Define attributes, data classes, optionality, and source mappings.
4. Propose business keys and surrogate keys.
5. Define relationships, cardinality, and optionality.
6. Define the history behavior for each entity.
7. Mark attributes affected by source-fit gaps.

Pattern guidance:

    Dimensional model   Reporting and analytics consumption
    Normalized model    Integrated core data shared by many consumers
    Hybrid layers       Integrated core with dimensional consumption marts

Key guidance:

- Use a stable business key to identify the real-world record.
- Propose a surrogate key when sources conflict, keys can change, or history requires one.
- Keep the source system identifier when business keys overlap across systems.

Definition conflict record:

    conflict_id: DC-001
    term: Customer
    definitions:
        - source: <SOURCE_A>
          meaning: <SOURCE_A_DEFINITION>
        - source: <SOURCE_B>
          meaning: <SOURCE_B_DEFINITION>
    affected_objects:
        - <MODEL_OBJECT>
    recommendation: <PROPOSED_DEFINITION>
    decision_owner: Lead Architect

Logical entity specification:

    entity: CUSTOMER
    definition: <BUSINESS_DEFINITION>
    grain: One row per customer version
    business_key:
        - SOURCE_SYSTEM
        - CUSTOMER_ID
    surrogate_key: CUSTOMER_KEY
    history: TYPE_2
    requirement_refs:
        - <REQUIREMENT_ID>
    attributes:
        - name: CUSTOMER_NAME
          data_class: TEXT
          nullable: false
          source: <SOURCE_COLUMN>
    relationships:
        - to: <RELATED_ENTITY>
          cardinality: ONE_TO_MANY
          optional: false

## Coding Conventions

- Use singular entity names, such as `CUSTOMER`.
- Use `_KEY` for surrogate keys and `_ID` for source identifiers.
- Use one approved name for one business concept.
- Keep Snowflake-specific details out of conceptual and logical specifications.
- Never merge conflicting definitions silently.
- Document every many-to-many relationship and propose how to resolve it.