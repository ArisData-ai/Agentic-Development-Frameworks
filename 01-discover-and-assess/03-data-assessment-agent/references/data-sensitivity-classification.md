---
name: data-sensitivity-classification
description: This skill defines how Snowflake Horizon classifies candidate source columns, applies system classification tags, and records sensitivity findings that feed risk scoring and early compliance review. Use this skill for any task involving sensitive data discovery, PII detection, PHI or financial field review, SYSTEM$CLASSIFY, semantic and privacy category tags, tag verification, or flagging sensitive fields in migration sources.
---

# Data Sensitivity Classification

## Overview

This skill identifies sensitive fields early so compliance teams can review them before data reaches the pipeline. It classifies and tags source columns, then records the findings used for risk scoring. Masking and access policies remain the responsibility of the Governance Agent.

## When to Use

- Classify candidate tables during an assessment run.
- Review or verify system classification tags.
- Add sensitivity findings to a readiness assessment.
- Prepare sensitive field lists for compliance teams.

## Instructions

1. Confirm that the account supports sensitive data classification and that the assessment role has the required privileges, including the SNOWFLAKE.CLASSIFICATION_ADMIN database role.
2. Classify the object and apply system tags.

        CALL SYSTEM$CLASSIFY(
            'SRC_DB.SALES.CUSTOMERS',
            {'auto_tag': true, 'sample_count': 10000}
        );

3. Retrieve the classification result.

        SELECT SYSTEM$GET_CLASSIFICATION_RESULT('SRC_DB.SALES.CUSTOMERS');

4. Verify the tags applied to the object.

        SELECT COLUMN_NAME, TAG_NAME, TAG_VALUE
        FROM TABLE(
            SRC_DB.INFORMATION_SCHEMA.TAG_REFERENCES_ALL_COLUMNS(
                'SRC_DB.SALES.CUSTOMERS', 'TABLE'
            )
        )
        WHERE TAG_NAME IN ('SEMANTIC_CATEGORY', 'PRIVACY_CATEGORY');

5. Store each finding in ASSESSMENT.SENSITIVITY_FINDINGS with RUN_ID, SOURCE_ID, COLUMN_NAME, SEMANTIC_CATEGORY, PRIVACY_CATEGORY, and CONFIDENCE.
6. Flag low-confidence results and any columns with descriptive names but no classification for manual review.
7. Record classification failures as UNKNOWN; never treat a failed classification as non-sensitive.
8. Send sensitive fields to the Lead Validator and to the Governance Agent as design input after validator review.

## Coding Conventions

- Store column names and categories only; never store sample values.
- Keep classification results linked to RUN_ID for auditability.
- Use parameterized object names and validate them against the assessment queue.
- Keep the classification stage separate from policy creation.
- Treat ambiguous results as sensitive until a reviewer confirms otherwise.