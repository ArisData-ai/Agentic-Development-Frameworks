---
name: pipeline-transformation-orchestration
description: This skill defines how the Pipeline Agent assembles incremental Snowflake transformation flows, using Dynamic Tables and Streams and Tasks to produce layered data objects, dependencies, schedules, and failure-aware orchestration from approved mappings. Use this skill for any task involving ELT transformations, Dynamic Tables, target lag, streams, tasks, task graphs, incremental loads, MERGE logic, pipeline dependencies, schedules, or curated data layers.
---

# Pipeline Transformation Orchestration

## Overview
This skill covers the path from raw data to curated data. It assembles approved transformation logic into incremental, dependency-aware Snowflake objects. SQL generation and legacy logic conversion remain the responsibility of the Code Generation Agent.

## When to Use
- A pipeline needs raw-to-curated transformation layers.
- A team must select Dynamic Tables or Streams and Tasks.
- Dependencies or schedules need to be defined.
- An approved mapping changes transformation logic or freshness requirements.

## Instructions
1. **Map the flow.** Identify source objects, target layers, dependencies, and the freshness requirement from the manifest.
2. **Select the pattern.**
    - Use **Dynamic Tables** for declarative transformations where Snowflake can manage refresh dependencies against a target lag.
    - Use **Streams and Tasks** for explicit procedural control, including conditional execution, custom `MERGE` behavior, or multi-step processing.
3. **Add approved logic.** Use only rules contained in the mapping or supplied by the Code Generation Agent. Flag undefined logic.
4. **Set dependencies.** Use Dynamic Table references or `AFTER` task relationships. Avoid overlapping schedules as a substitute for dependencies.
5. **Set freshness and schedules.** Align target lag and task schedules with the approved requirement. Do not refresh more often than required.
6. **Define failure behavior.** Specify how failures are surfaced and recovered. Ensure reruns do not create duplicates.
7. **Validate in non-production.** Confirm successful creation, dependency order, incremental behavior, and rerun safety. Leave production activation to the approved release process.

## Coding Conventions
- Use deterministic keys for incremental merges.
- Parameterize database, schema, and warehouse names per environment.
- Prefer explicit column lists over `SELECT *`.
- Use `TARGET_LAG = DOWNSTREAM` for intermediate Dynamic Tables when consumers set the required freshness.
- Use this Dynamic Table pattern:

        CREATE OR REPLACE DYNAMIC TABLE CURATED_DB.SALES.SALES_ORDERS
            TARGET_LAG = '1 hour'
            WAREHOUSE = TRANSFORM_WH
        AS
        SELECT
            ORDER_ID,
            CUSTOMER_ID,
            ORDER_DATE,
            NET_VALUE
        FROM RAW_DB.SAP.SALES_ORDERS;

- Use this Streams and Tasks pattern:

        CREATE OR REPLACE STREAM RAW_DB.SAP.SALES_ORDERS_STREAM
            ON TABLE RAW_DB.SAP.SALES_ORDERS;

        CREATE OR REPLACE TASK CURATED_DB.SALES.MERGE_SALES_ORDERS
            WAREHOUSE = TRANSFORM_WH
            SCHEDULE = '60 MINUTES'
            WHEN SYSTEM$STREAM_HAS_DATA('RAW_DB.SAP.SALES_ORDERS_STREAM')
        AS
            MERGE INTO CURATED_DB.SALES.SALES_ORDERS AS tgt
            USING RAW_DB.SAP.SALES_ORDERS_STREAM AS src
                ON tgt.ORDER_ID = src.ORDER_ID
            WHEN MATCHED THEN UPDATE SET
                tgt.CUSTOMER_ID = src.CUSTOMER_ID,
                tgt.ORDER_DATE = src.ORDER_DATE,
                tgt.NET_VALUE = src.NET_VALUE
            WHEN NOT MATCHED THEN INSERT (
                ORDER_ID,
                CUSTOMER_ID,
                ORDER_DATE,
                NET_VALUE
            )
            VALUES (
                src.ORDER_ID,
                src.CUSTOMER_ID,
                src.ORDER_DATE,
                src.NET_VALUE
            );

- Treat the `MERGE` example as a starting pattern. Add delete and duplicate-change handling when the source behavior requires it.