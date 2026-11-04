---
name: data-model-source-fit
description: This skill defines how the Data Modelling Agent validates approved requirements and assessed sources, tests grain, key, coverage, and history support with Snowflake SQL, and produces a source-fit findings report that flags design gaps. Use this skill for any task involving checking model readiness, profiling candidate keys, verifying grain, testing join coverage across SAP, Oracle, NetSuite, or Salesforce sources, identifying unsupported attributes, or explaining why source data cannot support a proposed design.
---

# Data Model Source Fit

## Overview

This skill determines whether the assessed sources can support the intended target model. It finds design blockers early and turns them into evidence for the Lead Architect.

## When to Use

Use this skill when:

- A modeling run starts.
- A requirement or source assessment changes.
- A proposed entity, grain, key, or history rule needs evidence.
- A reviewer questions whether the source data supports the design.

## Instructions

1. Confirm that each requirement has Lead Validator approval.
2. Confirm that each source has an accepted Data Assessment result. Carry forward its readiness score and known risks.
3. Build a coverage matrix that maps each required entity and attribute to candidate sources.
4. Test the grain, candidate keys, cross-source joins, and available history.
5. Classify each requirement using the statuses below.
6. Record each gap with its affected model objects, evidence, and options for the Lead Architect.

Fit statuses:

    SUPPORTED                 Source data meets the requirement as is
    SUPPORTED_WITH_TRANSFORM  Requirement can be met through documented logic
    PARTIAL                   Requirement can be met only for part of the scope
    UNSUPPORTED               Source data cannot meet the requirement

Candidate key test:

    SELECT
        COUNT(*)                     AS TOTAL_ROWS,
        COUNT(DISTINCT <KEY_COLUMN>) AS DISTINCT_KEYS,
        COUNT_IF(<KEY_COLUMN> IS NULL) AS NULL_KEYS
    FROM <SOURCE_TABLE>;

Join coverage test:

    SELECT
        COUNT(*)                                 AS CHILD_ROWS,
        COUNT_IF(P.<PARENT_KEY> IS NULL)         AS UNMATCHED_ROWS,
        DIV0(COUNT_IF(P.<PARENT_KEY> IS NULL), COUNT(*)) AS UNMATCHED_RATIO
    FROM <CHILD_TABLE> C
    LEFT JOIN <PARENT_TABLE> P
        ON C.<CHILD_KEY> = P.<PARENT_KEY>;

Findings record:

    finding_id: SF-001
    requirement_id: <REQUIREMENT_ID>
    model_object: <ENTITY_OR_ATTRIBUTE>
    source: <SOURCE_OBJECT>
    status: PARTIAL
    evidence: <MEASURED_RESULT>
    impact: <DESIGN_IMPACT>
    options:
        - <OPTION_A>
        - <OPTION_B>
    decision_owner: Lead Architect

## Coding Conventions

- Keep profiling queries read-only.
- Report measured results instead of general statements.
- Return aggregates in findings, not sensitive row-level values.
- Do not treat a passing sample as proof for the full source.
- Do not invent thresholds. Use the approved project values, or flag the threshold as missing.
- Do not resolve a gap by dropping a requirement.