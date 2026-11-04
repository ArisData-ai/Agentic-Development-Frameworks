---
name: architecture-reference-design
description: This skill defines how to convert selected options into a Snowflake reference architecture covering source connectivity, ingestion, incremental processing, storage layers, consumption, and operational boundaries, with Openflow and Streams and Tasks as core integration patterns. Use this skill for any task involving a reference architecture, solution design, integration blueprint, medallion design, CDC pattern, incremental processing design, ERP integration, or a Snowflake target-state architecture.
---

# Architecture Reference Design

## Overview

This skill produces a clear target-state architecture that delivery teams can build against. It defines system boundaries, data flow, integration patterns, service-level targets, and downstream responsibilities.

## When to Use

Use this skill after option evaluation identifies a preferred approach, or when an approved architecture needs a controlled revision. Do not use it to create detailed table models, final access policies, or production pipeline code.

## Instructions

1. Define the scope, including source systems, consumers, environments, regions, data domains, and exclusions.
2. Describe the end-to-end flow:
    - Sources and connectivity
    - Ingestion
    - Raw, curated, and consumption layers
    - Incremental processing
    - Consumers and interfaces
3. Use Openflow as the default candidate where it supports the source and meets the requirement. Document any alternative selected during evaluation.
4. Choose an incremental processing pattern that matches the required control and latency.
5. Set design targets for freshness, recovery, retention, concurrency, and growth.
6. Define failure handling, including retries, late-arriving data, backfills, replay, and schema changes.
7. Identify security boundaries and pass policy requirements to the Governance Agent.
8. Identify modeling implications and pass them to the Data Modelling Agent.
9. List build dependencies for the Pipeline Agent and Engineering Team.

Treat the following SQL as illustrative pattern references, not deployment-ready code.

Use a Dynamic Table when a declarative result and target lag meet the requirement:

    CREATE OR REPLACE DYNAMIC TABLE CURATED_DB.SALES.ORDER_SUMMARY
        TARGET_LAG = '15 minutes'
        WAREHOUSE = TRANSFORM_WH
    AS
    SELECT
        ORDER_DATE,
        REGION,
        SUM(ORDER_AMOUNT) AS TOTAL_AMOUNT
    FROM RAW_DB.SALES.ORDERS
    GROUP BY ORDER_DATE, REGION;

Use Streams and Tasks when the design needs explicit merge logic:

    CREATE OR REPLACE STREAM RAW_DB.SALES.ORDERS_STRM
        ON TABLE RAW_DB.SALES.ORDERS;

    CREATE OR REPLACE TASK CURATED_DB.SALES.MERGE_ORDERS_T
        WAREHOUSE = TRANSFORM_WH
        SCHEDULE = '5 MINUTES'
        WHEN SYSTEM$STREAM_HAS_DATA('RAW_DB.SALES.ORDERS_STRM')
    AS
        MERGE INTO CURATED_DB.SALES.ORDERS AS T
        USING (
            SELECT ORDER_ID, ORDER_STATUS, UPDATED_AT
            FROM RAW_DB.SALES.ORDERS_STRM
            WHERE METADATA$ACTION = 'INSERT'
        ) AS S
            ON T.ORDER_ID = S.ORDER_ID
        WHEN MATCHED THEN UPDATE SET
            T.ORDER_STATUS = S.ORDER_STATUS,
            T.UPDATED_AT = S.UPDATED_AT
        WHEN NOT MATCHED THEN INSERT
            (ORDER_ID, ORDER_STATUS, UPDATED_AT)
            VALUES (S.ORDER_ID, S.ORDER_STATUS, S.UPDATED_AT);

Add delete handling when the source can remove records.

Return the design in this structure:

    1. Scope and objectives
    2. Current-state constraints
    3. Target-state architecture
    4. Data flow by layer
    5. Integration patterns
    6. Service-level design targets
    7. Failure and recovery design
    8. Security and governance requirements
    9. Downstream dependencies
    10. Assumptions and open risks

## Coding Conventions

- Separate environments, domains, and processing layers.
- Use dedicated warehouses when workload isolation is a stated requirement.
- Express lag, retention, and recovery targets as measurable values.
- Label inferred details as assumptions.
- Keep sample SQL generic, and never include credentials or customer data.