---
name: governance-masking-row-access-policies
description: This skill defines the data protection policy process, which converts classifications, role models, and entitlement rules into Snowflake Horizon masking policies, row access policies, tag-based controls, and validation tests. Use this skill for any task involving dynamic data masking, row-level security, tag-based masking, PII or PHI protection, entitlement mapping tables, policy attachment, POLICY_CONTEXT testing, or requests to hide, restrict, or filter sensitive Snowflake data by role, region, tenant, or business unit.
---

# Governance Masking and Row Access Policies

## Overview

This skill defines column-level and row-level controls that protect sensitive data while preserving approved analytical use. Policies are reusable, testable, and tied to documented classifications and entitlements.

## When to Use

- Mask sensitive columns based on role or data sensitivity.
- Restrict rows by region, tenant, department, or business unit.
- Apply consistent protection through classification tags.
- Test policy behavior before Lead Architect approval.

## Instructions

1. **Select the control type.** Match each requirement to the narrowest effective control:
    - Use a masking policy to protect column values.
    - Use a row access policy to limit visible records.
    - Use tag-based masking to protect columns consistently by classification.

2. **Define the masking policy.** Create policies that match the protected data type:

        CREATE MASKING POLICY IF NOT EXISTS GOVERNANCE_DB.POLICIES.MP_MASK_STRING
            AS (VAL STRING) RETURNS STRING ->
            CASE
                WHEN IS_ROLE_IN_SESSION('FR_PII_APPROVED') THEN VAL
                ELSE '***MASKED***'
            END;

3. **Use tag-based masking when appropriate.** Attach the policy to an approved tag:

        ALTER TAG GOVERNANCE_DB.TAGS.DATA_SENSITIVITY
            SET MASKING POLICY GOVERNANCE_DB.POLICIES.MP_MASK_STRING;

4. **Define row-level entitlements.** Store entitlements in a governed mapping table:

        CREATE TABLE IF NOT EXISTS GOVERNANCE_DB.POLICIES.REGION_ENTITLEMENTS (
            ROLE_NAME STRING NOT NULL,
            REGION_CODE STRING NOT NULL
        );

5. **Define the row access policy.** Reference the mapping table rather than embedding entitlement values:

        CREATE ROW ACCESS POLICY IF NOT EXISTS GOVERNANCE_DB.POLICIES.RAP_REGION
            AS (REGION_CODE_VALUE STRING) RETURNS BOOLEAN ->
            EXISTS (
                SELECT 1
                FROM GOVERNANCE_DB.POLICIES.REGION_ENTITLEMENTS E
                WHERE E.REGION_CODE = REGION_CODE_VALUE
                  AND IS_ROLE_IN_SESSION(E.ROLE_NAME)
            );

6. **Stage policy attachments.** Include the deployment statement in the review package, but do not run it before approval:

        ALTER TABLE ANALYTICS_DB.SALES.FACT_ORDERS
            ADD ROW ACCESS POLICY GOVERNANCE_DB.POLICIES.RAP_REGION
            ON (REGION_CODE);

7. **Test the policy.** Validate approved, denied, and edge-case roles:

        EXECUTE USING POLICY_CONTEXT(CURRENT_ROLE => 'FR_CUSTOMER_ANALYST')
            AS SELECT EMAIL FROM ANALYTICS_DB.CUSTOMER.DIM_CUSTOMER;

8. **Assess design impact.** Document effects on joins, aggregations, downstream views, query performance, shared data, and operational workloads.

## Coding Conventions

- Prefix masking policies with `MP_` and row access policies with `RAP_`.
- Keep policies in a dedicated governance database and schema.
- Use `IS_ROLE_IN_SESSION` for role-hierarchy-aware checks.
- Default to the most restrictive result when access conditions are not met.
- Keep policy logic simple and store changing entitlements in mapping tables.
- Use a separate policy administration role.
- Include expected results for each role in the test plan.