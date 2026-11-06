---
name: legacy-code-conversion
description: This skill defines how to convert legacy SQL, stored procedures, and database scripts into Snowflake-compatible code while preserving business behavior, rating translation confidence, and flagging statements that require engineering judgment. Use this skill for any task involving migrating, translating, or refactoring Oracle PL/SQL, SQL Server T-SQL, Teradata BTEQ, DB2, Netezza, or SAP-backed database logic, converting stored procedures to Snowflake Scripting or Snowpark, or reviewing dialect differences.
---

# Legacy Code Conversion

## Overview

This skill converts legacy database logic into Snowflake-compatible code. It preserves approved business behavior, avoids syntax-only conversions that change results, and flags any statement that the agent cannot translate with confidence.

## When to Use

- Legacy SQL or stored procedures are approved for migration.
- A dialect-specific function, data type, or procedural pattern requires conversion.
- An existing conversion needs a confidence assessment or reviewer revision.

Use `sql-transformation-generation` when the input is an approved design rather than existing code.

## Instructions

1. Inventory each legacy object, its dialect, dependencies, parameters, and side effects.
2. Identify the behavior that must be preserved, including null handling, transaction boundaries, date logic, and error handling.
3. Retrieve approved conversion patterns through `coding-standards-retrieval`.
4. Translate syntax and data types.
5. Replace row-by-row logic with set-based SQL when the behavior remains equivalent.
6. Use Snowflake Scripting or Snowpark for procedural logic that cannot be expressed cleanly in set-based SQL.
7. Rate each converted unit as HIGH, MEDIUM, or LOW confidence.
8. Keep the original statement reference beside any converted logic with a REVIEW or BLOCKER flag.
9. Provide a conversion summary with counts by confidence level and open flag.

### Confidence Ratings

- **HIGH:** The conversion is direct, and the expected behavior is equivalent. No flag is required.
- **MEDIUM:** The code is complete, but a reviewer must confirm a behavior difference or assumption. Add a `REVIEW` flag.
- **LOW:** The agent cannot translate the logic with confidence. Add a `BLOCKER` flag and do not provide guessed executable logic.

### Patterns That Require Review

- **Oracle packages:** Split into schema-level procedures and functions, and review package state.
- **Triggers:** Assess Streams and Tasks or application-level alternatives.
- **Cursors:** Prefer set-based logic when ordering and side effects permit it.
- **Dynamic SQL:** Use `EXECUTE IMMEDIATE`, and validate object names and bind values.
- **Transactions and exception handling:** Confirm rollback, commit, and error behavior.
- **External calls:** Flag file, network, messaging, and scheduler dependencies.
- **Indexes and hints:** Remove them only after recording the original performance intent.

## Coding Conventions

Check these behavior differences before assigning HIGH confidence:

- **Empty strings:** Oracle treats an empty string as `NULL`, while Snowflake treats it as a string value.
- **Date and time:** Confirm time zone and return type. Snowflake `SYSDATE` returns UTC.
- **Division:** Teradata integer division can truncate, while Snowflake can return a decimal result.
- **String comparison:** Confirm case sensitivity and collation, especially for SQL Server sources.
- **Numeric precision:** Confirm precision, scale, and rounding.
- **Merge duplicates:** Deduplicate source rows so each target row has one intended match.
- **Identity values:** Do not assume gap-free sequences.

Use this conversion note format:

    -- CONVERTED FROM: <dialect>.<object_name>, lines <start>-<end>
    -- CONFIDENCE: <HIGH, MEDIUM, or LOW>
    -- BEHAVIOR NOTE: <difference or None>

Use this format for logic that cannot be translated with confidence:

    -- CONVERTED FROM: <dialect>.<object_name>, lines <start>-<end>
    -- CONFIDENCE: LOW
    -- CODEGEN-FLAG: <FLAG_ID> | BLOCKER | <reason and required decision>
    -- ORIGINAL STATEMENT:
    -- <legacy statement>