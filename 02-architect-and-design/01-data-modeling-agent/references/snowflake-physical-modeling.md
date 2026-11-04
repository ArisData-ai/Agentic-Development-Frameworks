---
name: snowflake-physical-modeling
description: This skill defines how the Data Modelling Agent translates approved logical models into Snowflake physical designs, producing schema layouts, table DDL, data types, constraints, clustering recommendations, and declarative Dynamic Table layers. Use this skill for any task involving writing CREATE TABLE statements, designing staging, core, or mart schemas, selecting Snowflake data types, defining informational keys, configuring TARGET_LAG, choosing between tables and Dynamic Tables, or preparing physical model specifications for pipeline and code generation teams.
---

# Snowflake Physical Modeling

## Overview

This skill converts the logical design into Snowflake design artifacts. The output gives downstream engineers clear, consistent structures to build against. All generated DDL remains a proposal until the Lead Architect approves it.

## When to Use

Use this skill when:

- Translating a logical model into Snowflake objects.
- Choosing between a standard table and a Dynamic Table.
- Defining data types, keys, or clustering.
- Designing declarative transformation or consumption layers.

## Instructions

1. Map each logical entity to a target schema and object type.
2. Preserve the documented grain, keys, and history behavior.
3. Assign Snowflake data types from the source evidence and business meaning.
4. Declare primary and foreign keys as model metadata.
5. Use Dynamic Tables where a declarative query and a target freshness can define the layer.
6. Recommend clustering only with evidence of large-table pruning needs.
7. Generate DDL for review, and list any object that depends on an open decision.

Object selection:

    Standard table   Stored history, externally loaded data, or custom merge logic
    Dynamic Table    Declarative transformation with an agreed target freshness
    View             Lightweight logic with no stored result required

Table DDL pattern:

    CREATE TABLE IF NOT EXISTS <DB>.<CORE_SCHEMA>.DIM_CUSTOMER (
        CUSTOMER_KEY    NUMBER(38,0)  NOT NULL,
        SOURCE_SYSTEM   STRING        NOT NULL,
        CUSTOMER_ID     STRING        NOT NULL,
        CUSTOMER_NAME   STRING        NOT NULL,
        EFFECTIVE_FROM  TIMESTAMP_NTZ NOT NULL,
        EFFECTIVE_TO    TIMESTAMP_NTZ,
        IS_CURRENT      BOOLEAN       NOT NULL,
        CONSTRAINT PK_DIM_CUSTOMER PRIMARY KEY (CUSTOMER_KEY)
    )
    COMMENT = 'One row per customer version';

Dynamic Table pattern:

    CREATE OR REPLACE DYNAMIC TABLE <DB>.<MART_SCHEMA>.FCT_DAILY_SALES
        TARGET_LAG = '<APPROVED_LAG>'
        WAREHOUSE = <TRANSFORM_WH>
        REFRESH_MODE = AUTO
        COMMENT = 'One row per product, customer, and day'
    AS
    SELECT
        ORDER_DATE,
        PRODUCT_KEY,
        CUSTOMER_KEY,
        SUM(NET_AMOUNT) AS NET_AMOUNT
    FROM <DB>.<CORE_SCHEMA>.FCT_ORDER_LINE
    GROUP BY
        ORDER_DATE,
        PRODUCT_KEY,
        CUSTOMER_KEY;

Layer dependency pattern:

    Intermediate Dynamic Table   TARGET_LAG = DOWNSTREAM
    Final Dynamic Table          TARGET_LAG = '<APPROVED_LAG>'

## Coding Conventions

- Prefix facts with `FCT_` and dimensions with `DIM_`.
- Add a `COMMENT` that states the grain of every table.
- Use `TIMESTAMP_NTZ` only when the time zone rule is documented.
- Use `NUMBER` with explicit precision and scale for monetary and quantity fields.
- Do not rely on `PRIMARY KEY`, `UNIQUE`, or `FOREIGN KEY` to reject bad data in standard tables, because Snowflake does not enforce them. Route validation needs to the Data Quality Agent.
- Do not set a warehouse or `TARGET_LAG` by assumption. Use a placeholder until the project value is confirmed.
- Leave roles, masking, and row-access policies to the Governance Agent.