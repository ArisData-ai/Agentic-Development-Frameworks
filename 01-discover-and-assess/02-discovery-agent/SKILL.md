---
name: discovery-agent-orchestration
description: This skill defines the end-to-end orchestration of a Discovery Agent that takes source system scopes and sweep schedules as inputs and produces a governed inventory, dependency map, and validation package through Cortex Agents, Openflow, and Snowflake Horizon. Use this skill for any task involving building, configuring, scheduling, running, or troubleshooting a Discovery Agent, including recurring discovery sweeps, legacy estate mapping, migration scoping, sweep state tracking, exception handling, and coordination across Teradata, Oracle, SQL Server, SAP, Informatica, SSIS, Airflow, and Control-M environments.
---

# Discovery Agent Orchestration

## Overview

The Discovery Agent continuously maps the legacy systems, sources, and dependencies that a data initiative must account for. It replaces manual system inventories at the start of migration and modernization programs.

This skill defines the control flow across the agent's Sense, Act, and Hand Off phases. The agent recommends findings, but the Lead Validator owns the final scope decision.

## When to Use

- Build or configure a Discovery Agent.
- Schedule recurring discovery sweeps.
- Add source systems to an existing discovery program.
- Coordinate extraction, cataloging, dependency mapping, and validation.
- Troubleshoot failed or partial sweeps.

## Instructions

1. Register each in-scope source with its owner, technology, environment, connection method, and sweep frequency.
2. Create a unique sweep record before any extraction begins.
3. Run the phases in this order:
    - Extract metadata and detect changes.
    - Catalog new and changed assets.
    - Trace dependencies and flag undocumented or orphaned components.
    - Build the Lead Validator package.
4. Record the status of each source and phase so a failed sweep can resume without repeating completed work.
5. Isolate failures to the affected source, continue with the remaining sources, and mark the sweep as partial.
6. Never report a source as complete if extraction was partial or permissions were missing.
7. Release findings to the Data Assessment Agent only after the Lead Validator confirms scope.

Use a control table to track sweep state:

    CREATE TABLE IF NOT EXISTS DISCOVERY_DB.CONTROL.DISCOVERY_SWEEP (
        SWEEP_ID            STRING NOT NULL,
        SOURCE_SYSTEM_ID    STRING NOT NULL,
        SWEEP_PHASE         STRING NOT NULL,
        SWEEP_STATUS        STRING NOT NULL,
        STARTED_AT          TIMESTAMP_NTZ,
        COMPLETED_AT        TIMESTAMP_NTZ,
        ERROR_DETAIL        STRING,
        CONSTRAINT PK_DISCOVERY_SWEEP
            PRIMARY KEY (SWEEP_ID, SOURCE_SYSTEM_ID, SWEEP_PHASE)
    );

Use these status values: `PENDING`, `RUNNING`, `COMPLETED`, `PARTIAL`, `FAILED`, and `SKIPPED`.

Schedule the sweep through a Snowflake Task:

    CREATE OR REPLACE TASK DISCOVERY_DB.CONTROL.TSK_DISCOVERY_SWEEP
        WAREHOUSE = DISCOVERY_WH
        SCHEDULE = 'USING CRON 0 2 * * 1 UTC'
    AS
        CALL DISCOVERY_DB.CONTROL.SP_RUN_DISCOVERY_SWEEP();

    ALTER TASK DISCOVERY_DB.CONTROL.TSK_DISCOVERY_SWEEP RESUME;

## Coding Conventions

- Use uppercase Snowflake identifiers with clear `RAW`, `CONTROL`, `CATALOG`, `LINEAGE`, and `VALIDATION` schema boundaries.
- Make every sweep step idempotent and keyed by `SWEEP_ID`.
- Parameterize source, environment, warehouse, and schedule values.
- Store secrets in Snowflake secrets, never in code or prompts.
- Grant the agent role read-only metadata access and only the required write access in `DISCOVERY_DB`.
- Log every failure with the source, phase, timestamp, and error detail.

## References

- `references/discovery-source-metadata-extraction.md`: Connects to source systems, captures metadata snapshots, and detects changes.
- `references/discovery-asset-cataloging.md`: Standardizes assets and stores governed catalog records.
- `references/discovery-dependency-mapping.md`: Traces dependencies and flags undocumented or orphaned components.
- `references/discovery-validation-handoff.md`: Packages findings, supports Lead Validator review, and releases confirmed scope.