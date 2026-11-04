---
name: governance-compliance-coverage-reporting
description: This skill defines the governance validation and reporting process, which compares designs against policy requirements, identifies control gaps, and produces violation findings, coverage metrics, and audit evidence through Cortex Agents, Cortex Analyst, and Snowflake Horizon. Use this skill for any task involving policy violation checks, least-privilege validation, masking or row access coverage, audit reporting, compliance evidence, lineage review, ACCOUNT_USAGE analysis, or requests to report, verify, or explain governance readiness across a Snowflake design.
---

# Governance Compliance and Coverage Reporting

## Overview

This skill checks whether proposed designs meet governance requirements and reports the coverage of approved controls. It gives the Lead Architect, security teams, data owners, and auditors a consistent view of gaps and decisions.

## When to Use

- Validate a design before Lead Architect review.
- Monitor design changes against governance policy.
- Report on classification, masking, row access, or role coverage.
- Prepare documented evidence for audit or compliance review.

## Instructions

1. **Define validation rules.** Convert policy requirements into testable checks, including:
    - Sensitive columns have approved tags.
    - Restricted data has an approved protection control.
    - Roles follow least-privilege standards.
    - Sensitive data paths have an identified owner.
    - Exceptions include a reason, approver, and review date.

2. **Compare design and policy.** Evaluate proposed objects, roles, grants, lineage, and data flows. Flag any design that creates an uncontrolled access path.

3. **Assign severity.** Use a consistent scale:
    - **Critical:** Sensitive data is exposed or a required control is missing.
    - **High:** Access exceeds the approved business need.
    - **Medium:** A control exists but is incomplete or inconsistently applied.
    - **Low:** Documentation or naming does not meet standards.

4. **Measure policy coverage.** Query Horizon governance metadata and account usage views:

        SELECT
            T.OBJECT_DATABASE,
            T.OBJECT_SCHEMA,
            T.OBJECT_NAME,
            T.COLUMN_NAME,
            T.TAG_VALUE,
            P.POLICY_NAME
        FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES T
        LEFT JOIN SNOWFLAKE.ACCOUNT_USAGE.POLICY_REFERENCES P
            ON T.OBJECT_DATABASE = P.REF_DATABASE_NAME
           AND T.OBJECT_SCHEMA = P.REF_SCHEMA_NAME
           AND T.OBJECT_NAME = P.REF_ENTITY_NAME
           AND T.COLUMN_NAME = P.REF_COLUMN_NAME
        WHERE T.TAG_NAME = 'DATA_SENSITIVITY'
          AND T.TAG_VALUE IN ('CONFIDENTIAL', 'RESTRICTED')
          AND T.COLUMN_NAME IS NOT NULL;

5. **Report coverage through Cortex Analyst.** Expose approved metrics for natural language reporting, including classified column coverage, protected sensitive column coverage, open violations, exception count, and decision status.

6. **Prepare the findings report.** For each finding, include the affected object, violated requirement, severity, evidence, recommended action, owner, and status.

7. **Submit results for review.** Include open violations and coverage results in the policy set. Do not mark a finding resolved until the required action or approved exception is recorded.

## Coding Conventions

- Use deterministic rules for compliance checks wherever possible.
- Keep rule IDs stable across reporting periods.
- Include the evaluation timestamp in every result.
- Account for metadata latency when using `ACCOUNT_USAGE`.
- Report counts, percentages, and object references without exposing sensitive values.
- Retain decisions, exceptions, and remediation history for audit.