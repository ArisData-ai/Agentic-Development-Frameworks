---
name: data-source-profiling
description: This skill defines how Snowpark profiling logic measures completeness, consistency, and volume for candidate tables and records column-level metrics and rule failures for readiness scoring. Use this skill for any task involving data profiling, null analysis, duplicate checks, distinct counts, format validation, row count checks, Snowpark profiling procedures, or source quality metrics for Oracle, SQL Server, Teradata, SAP, or Salesforce data.
---

# Data Source Profiling

## Overview

This skill runs profiling logic close to the data with Snowpark. It produces repeatable metrics for completeness, consistency, and volume without moving or changing source data.

## When to Use

- Build or update Snowpark profiling procedures.
- Measure null rates, distinct counts, duplicates, and row counts.
- Add consistency rules for keys, formats, or accepted values.
- Investigate unexpected profiling metrics.

## Instructions

1. Read the object name and RUN_ID from the assessment workflow.
2. Calculate table-level and column-level metrics in one aggregate query.

        from snowflake.snowpark import Session
        from snowflake.snowpark import functions as F

        def profile_table(session: Session, run_id: str, source_id: str, object_fqn: str) -> int:
            df = session.table(object_fqn)
            fields = df.schema.fields

            metrics = [F.count(F.lit(1)).alias("ROW_COUNT")]
            for i, field in enumerate(fields):
                metrics.append(F.count(F.col(field.name)).alias(f"NON_NULL_{i}"))
                metrics.append(F.approx_count_distinct(F.col(field.name)).alias(f"DISTINCT_{i}"))

            result = df.agg(*metrics).collect()[0]
            row_count = result["ROW_COUNT"]

            rows = []
            for i, field in enumerate(fields):
                non_null = result[f"NON_NULL_{i}"]
                null_pct = 0.0 if row_count == 0 else round(100 * (row_count - non_null) / row_count, 2)
                rows.append((
                    run_id, source_id, field.name, str(field.datatype),
                    row_count, null_pct, result[f"DISTINCT_{i}"]
                ))

            if rows:
                session.create_dataframe(
                    rows,
                    schema=[
                        "RUN_ID", "SOURCE_ID", "COLUMN_NAME", "DATA_TYPE",
                        "ROW_COUNT", "NULL_PCT", "DISTINCT_COUNT"
                    ]
                ).write.mode("append").save_as_table(
                    "ASSESSMENT.PROFILE_RESULTS", column_order="name"
                )

            return row_count

3. Run consistency rules and store each result in ASSESSMENT.RULE_RESULTS with RECORDS_CHECKED and RECORDS_FAILED counts.

        SELECT
            COUNT(*) AS RECORDS_CHECKED,
            COUNT(*) - COUNT(DISTINCT CUSTOMER_ID) AS RECORDS_FAILED
        FROM SRC_DB.SALES.CUSTOMERS;

4. Apply these checks where the required metadata exists:
    - Key uniqueness for declared or candidate keys.
    - Format conformance for dates, emails, and identifiers.
    - Accepted values for coded fields.
    - Referential integrity between related tables.
5. Compare the row count to the expected range supplied by discovery metadata.
6. Record any check that could not run as NOT_EVALUATED; never count it as a pass.

## Coding Conventions

- Keep profiling logic parameterized and reusable across sources.
- Push computation into Snowflake; do not collect full tables to the client.
- Use approximate distinct counts for large tables and label them as approximate.
- Record the sample rate whenever profiling uses a sample.
- Store metrics only; do not persist raw values in profiling results.