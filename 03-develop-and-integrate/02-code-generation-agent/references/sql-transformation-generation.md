---
name: sql-transformation-generation
description: This skill defines how to turn approved source-to-target mappings and transformation specifications into standards-aligned Snowflake SQL, stored procedures, and Snowpark transformation code with full traceability and explicit flags for unresolved logic. Use this skill for any task involving writing SQL from a mapping document, generating MERGE or CTAS statements, creating views, building Snowpark Python transformations, implementing business rules, or producing staging, integration, and presentation layer code.
---

# SQL and Transformation Generation

## Overview

This skill converts approved designs into Snowflake SQL or Snowpark code. It keeps generated logic consistent, readable, and traceable to the approved specification.

## When to Use

- A source-to-target mapping requires SQL.
- A transformation specification requires a view, table build, merge, or stored procedure.
- Complex or reusable logic is better suited to Snowpark.
- A reviewer requests changes to generated transformation code.

Use `legacy-code-conversion` when the input is existing legacy code.

## Instructions

1. Read the approved specification and record its ID and version.
2. Confirm the source objects, target objects, grain, keys, data types, load pattern, and business rules.
3. Retrieve applicable standards and approved patterns through `coding-standards-retrieval`.
4. Use SQL for set-based transformations that are clear in SQL.
5. Use Snowpark when the approved design calls for reusable Python logic or logic that is difficult to maintain in SQL.
6. Implement only the rules in the specification.
7. Map each target column to its source column or approved transformation rule.
8. Add a `BLOCKER` flag when a required rule, join, key, or data type is missing or contradictory.
9. Add a `REVIEW` flag when the code relies on an assumption that a person must confirm.
10. Confirm that every target column is mapped or flagged before handoff.

## Coding Conventions

- Use uppercase SQL keywords and consistent object naming from the retrieved standards.
- Use fully qualified object names or approved environment variables.
- Use CTEs to separate source selection, deduplication, and transformation logic.
- List columns explicitly. Do not use `SELECT *` in production logic.
- Cast data types explicitly when the source and target types differ.
- Define the deduplication rule before using a `MERGE`.
- Use `TRY_` conversion functions when the specification requires invalid values to be handled without failing the load.
- Parameterize environment-specific values.
- Never hardcode credentials, tokens, or connection strings.

SQL merge pattern:

    MERGE INTO <TARGET_DB>.<TARGET_SCHEMA>.<TARGET_TABLE> AS tgt
    USING (
        WITH source_data AS (
            SELECT
                <BUSINESS_KEY>,
                <SOURCE_COLUMN>,
                <SOURCE_UPDATED_AT>
            FROM <SOURCE_DB>.<SOURCE_SCHEMA>.<SOURCE_TABLE>
        )
        SELECT
            <BUSINESS_KEY>,
            <TRANSFORMED_COLUMN_EXPRESSION> AS <TARGET_COLUMN>,
            <SOURCE_UPDATED_AT>
        FROM source_data
        QUALIFY ROW_NUMBER() OVER (
            PARTITION BY <BUSINESS_KEY>
            ORDER BY <SOURCE_UPDATED_AT> DESC
        ) = 1
    ) AS src
        ON tgt.<BUSINESS_KEY> = src.<BUSINESS_KEY>
    WHEN MATCHED AND src.<SOURCE_UPDATED_AT> > tgt.<SOURCE_UPDATED_AT> THEN
        UPDATE SET
            tgt.<TARGET_COLUMN> = src.<TARGET_COLUMN>,
            tgt.<SOURCE_UPDATED_AT> = src.<SOURCE_UPDATED_AT>
    WHEN NOT MATCHED THEN
        INSERT (<BUSINESS_KEY>, <TARGET_COLUMN>, <SOURCE_UPDATED_AT>)
        VALUES (src.<BUSINESS_KEY>, src.<TARGET_COLUMN>, src.<SOURCE_UPDATED_AT>);

Snowpark transformation pattern:

    from snowflake.snowpark import Session
    from snowflake.snowpark.functions import col, trim, upper

    def run(session: Session, source_table: str, target_table: str) -> str:
        transformed = (
            session.table(source_table)
            .filter(col("<BUSINESS_KEY>").is_not_null())
            .select(
                col("<BUSINESS_KEY>"),
                upper(trim(col("<SOURCE_COLUMN>"))).alias("<TARGET_COLUMN>")
            )
        )

        transformed.write.mode("append").save_as_table(target_table)
        return f"Loaded {target_table}"