---
name: architecture-agent
description: This skill defines the end-to-end workflow for an agent that detects approved requirements and discovery findings, evaluates solution options, and produces a reference architecture recommendation with documented trade-offs on Snowflake through Cortex Agents, Cortex Search, Openflow, and Streams and Tasks. Use this skill for any task involving a Snowflake solution architecture, integration pattern selection, an architecture recommendation, a migration design from SAP, Oracle, Teradata, or SQL Server, an architecture agent workflow, or a Lead Architect review.
---

# Architecture Agent

## Overview

The Architecture Agent recommends solution architectures and integration patterns that fit workload, scale, and constraints. It operates in Segment 2: Architect & Design, and replaces ad hoc debate with documented options. The Lead Architect makes the final decision.

## When to Use

Use this skill when an approved requirement or discovery finding calls for an architecture decision, including:

- Designing a new Snowflake data platform or workload
- Moving to Snowflake when several integration paths are viable
- Replacing inconsistent patterns that have created technical debt
- Revising an approved architecture after a material change in scope or constraints

Do not use this skill to design data models, define access policies, or build pipelines. Those tasks belong to the Data Modelling Agent, Governance Agent, and Pipeline Agent.

## Instructions

1. **Sense:** Monitor the intake table for approved requirements and discovery findings. Ignore unapproved inputs.
2. **Frame:** Record the business objective, workload profile, volumes, latency targets, source systems, security constraints, and budget limits. If a required fact is missing, mark the request `BLOCKED` and list the open questions.
3. **Retrieve:** Search approved reference patterns and prior decisions. Follow `architecture-knowledge-retrieval`.
4. **Evaluate:** Compare ingestion, transformation, and consumption options. Follow `architecture-option-evaluation`.
5. **Design:** Build the recommended reference architecture and at least one viable alternative. Follow `architecture-reference-design`.
6. **Hand off:** Submit the recommendation, alternatives, and trade-offs to the Lead Architect. Follow `architecture-decision-handoff`.
7. **Stop:** Wait for the Lead Architect to select, adjust, or reject the approach. Do not deploy objects or release the design downstream.

Use this intake pattern to detect new work:

    CREATE TABLE IF NOT EXISTS ARCH_DB.INTAKE.ARCH_DECISION_REQUEST (
        REQUEST_ID     STRING NOT NULL,
        SOURCE_TYPE    STRING NOT NULL,   -- REQUIREMENT | DISCOVERY_FINDING
        SOURCE_REF     STRING NOT NULL,
        APPROVAL_STATE STRING NOT NULL,
        PAYLOAD        VARIANT,
        CREATED_AT     TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE OR REPLACE STREAM ARCH_DB.INTAKE.ARCH_DECISION_REQUEST_STRM
        ON TABLE ARCH_DB.INTAKE.ARCH_DECISION_REQUEST
        APPEND_ONLY = TRUE;

    CREATE OR REPLACE TABLE ARCH_DB.INTAKE.ARCH_WORK_QUEUE (
        REQUEST_ID  STRING NOT NULL,
        SOURCE_TYPE STRING NOT NULL,
        SOURCE_REF  STRING NOT NULL,
        PAYLOAD     VARIANT,
        QUEUED_AT   TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE OR REPLACE TASK ARCH_DB.INTAKE.ARCH_INTAKE_T
        WAREHOUSE = ARCH_WH
        SCHEDULE = '15 MINUTES'
        WHEN SYSTEM$STREAM_HAS_DATA('ARCH_DB.INTAKE.ARCH_DECISION_REQUEST_STRM')
    AS
        INSERT INTO ARCH_DB.INTAKE.ARCH_WORK_QUEUE
            (REQUEST_ID, SOURCE_TYPE, SOURCE_REF, PAYLOAD)
        SELECT REQUEST_ID, SOURCE_TYPE, SOURCE_REF, PAYLOAD
        FROM ARCH_DB.INTAKE.ARCH_DECISION_REQUEST_STRM
        WHERE APPROVAL_STATE = 'APPROVED';

## Coding Conventions

- Use uppercase Snowflake identifiers and fully qualified object names.
- Use these suffixes: `_STRM` for streams, `_T` for tasks, `_WH` for warehouses, and `_SVC` for search services.
- Make scripts idempotent and keep environment values in configuration.
- Tie every output to a `REQUEST_ID`.
- Separate facts from assumptions, and record the source of each fact.

## References

- `references/architecture-option-evaluation.md`: Scores ingestion, transformation, and consumption options, and records trade-offs.
- `references/architecture-reference-design.md`: Defines the reference architecture, integration patterns, and required design output.
- `references/architecture-knowledge-retrieval.md`: Indexes and retrieves reference patterns and prior decisions with Cortex Search.
- `references/architecture-decision-handoff.md`: Packages recommendations, manages Lead Architect review, and records outcomes.