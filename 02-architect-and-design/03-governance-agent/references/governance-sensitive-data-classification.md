---
name: governance-sensitive-data-classification
description: This skill defines the sensitive data classification process, which inspects schemas, columns, and design metadata, proposes semantic and sensitivity tags through Snowflake Horizon, and produces a reviewed classification inventory for access control design. Use this skill for any task involving PII, PHI, PCI, or financial data discovery, Horizon classification, SYSTEM$CLASSIFY runs, tag taxonomy design, column tagging, or requests to find, label, or review sensitive fields from CRM, HR, SAP, or Oracle ERP sources.
---

# Governance Sensitive Data Classification

## Overview

This skill identifies sensitive data early and converts the findings into a governed tag inventory. Classification results drive role design, masking, row-level controls, and compliance reporting.

## When to Use

- Evaluate new sources, schemas, tables, or columns.
- Define or extend a sensitivity tag taxonomy.
- Review classification results before policy design.
- Reclassify data after a design or source change.

## Instructions

1. **Define the classification scope.** Identify the target databases, schemas, and objects. Include sensitivity indicators from approved requirements and data assessment findings.

2. **Run classification in proposal mode.** Generate results without applying tags automatically:

        CALL SYSTEM$CLASSIFY(
            'ANALYTICS_DB.CUSTOMER.DIM_CUSTOMER',
            {'auto_tag': false}
        );

3. **Supplement automated results.** Review column names, descriptions, source context, and business definitions. Flag free-text, semi-structured, and derived fields for manual review.

4. **Propose sensitivity tags.** Use a controlled taxonomy:

        CREATE TAG IF NOT EXISTS GOVERNANCE_DB.TAGS.DATA_SENSITIVITY
            ALLOWED_VALUES 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED'
            COMMENT = 'Enterprise data sensitivity classification';

5. **Stage tag assignments for review.** Present proposed assignments as deployable statements, but do not apply them before approval:

        ALTER TABLE ANALYTICS_DB.CUSTOMER.DIM_CUSTOMER
            MODIFY COLUMN EMAIL
            SET TAG GOVERNANCE_DB.TAGS.DATA_SENSITIVITY = 'CONFIDENTIAL';

6. **Escalate uncertain results.** Mark low-confidence or conflicting classifications for data owner review. Apply the stricter proposed classification until the issue is resolved.

7. **Publish the classification inventory.** Include the object, column, proposed tag, evidence, confidence, data owner, and review status.

## Coding Conventions

- Maintain tags in a dedicated governance database and schema.
- Use uppercase, descriptive tag names.
- Define `ALLOWED_VALUES` for controlled tags.
- Record classification evidence without copying sensitive values.
- Keep classification proposals separate from enforcement actions.
- Reassess classifications when source structures or business usage change.