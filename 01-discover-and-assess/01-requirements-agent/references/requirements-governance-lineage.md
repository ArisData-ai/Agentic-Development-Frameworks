---
name: requirements-governance-lineage
description: This skill defines how the Requirements Agent governs source documents, sensitive content, and requirement lineage through Snowflake Horizon classification, object tags, masking policies, row access policies, and access history. Use this skill for any task involving requirements traceability, audit evidence, RBAC, program-level access control, sensitive data protection, source citation checks, compliance reviews, or objective-to-requirement lineage reporting.
---

# Requirements Governance and Lineage

## Overview

This skill keeps source documents governed and requirement lineage auditable. Every approved requirement must trace to its objective, source material, validation decision, and baseline version.

## When to Use

- Set up roles and access policies for requirements data.
- Protect sensitive content in intake material.
- Prove where a requirement came from.
- Produce audit or compliance evidence.

## Instructions

1. Define least-privilege roles.

        CREATE ROLE IF NOT EXISTS REQ_AGENT_SERVICE;
        CREATE ROLE IF NOT EXISTS REQ_LEAD_VALIDATOR;
        CREATE ROLE IF NOT EXISTS REQ_READER;
        CREATE ROLE IF NOT EXISTS REQ_AUDITOR;

2. Classify intake tables and apply sensitivity and program tags.

        CREATE TAG IF NOT EXISTS SENSITIVITY
            ALLOWED_VALUES 'PUBLIC', 'INTERNAL', 'CONFIDENTIAL', 'RESTRICTED';

        ALTER TABLE INTAKE_CHUNK
            SET TAG SENSITIVITY = 'CONFIDENTIAL';

3. Mask sensitive text for roles that do not need it.

        CREATE MASKING POLICY IF NOT EXISTS MASK_INTAKE_TEXT
            AS (VAL STRING) RETURNS STRING ->
            CASE
                WHEN IS_ROLE_IN_SESSION('REQ_LEAD_VALIDATOR') THEN VAL
                WHEN IS_ROLE_IN_SESSION('REQ_AGENT_SERVICE') THEN VAL
                ELSE '*** MASKED ***'
            END;

        ALTER TABLE INTAKE_CHUNK
            MODIFY COLUMN CHUNK_TEXT
            SET MASKING POLICY MASK_INTAKE_TEXT;

4. Apply a row access policy so users see only authorized programs.

        CREATE ROW ACCESS POLICY IF NOT EXISTS PROGRAM_ACCESS
            AS (PROGRAM_VALUE STRING) RETURNS BOOLEAN ->
            EXISTS (
                SELECT 1
                FROM PROGRAM_ROLE_MAP m
                WHERE m.PROGRAM_ID = PROGRAM_VALUE
                    AND IS_ROLE_IN_SESSION(m.ROLE_NAME)
            );

        ALTER TABLE REQUIREMENT
            ADD ROW ACCESS POLICY PROGRAM_ACCESS ON (PROGRAM_ID);

5. Store one row per requirement and source chunk in `REQUIREMENT_SOURCE`.
6. Publish a traceability view.

        CREATE OR REPLACE VIEW REQUIREMENT_TRACE_V AS
        SELECT
            r.PROGRAM_ID, r.REQUIREMENT_ID, r.VERSION, r.STATEMENT,
            o.OBJECTIVE_TEXT, d.DOCUMENT_ID, d.SOURCE_TYPE, s.CHUNK_ID,
            v.DECISION, v.DECIDED_BY, v.DECIDED_AT
        FROM REQUIREMENT r
        LEFT JOIN BUSINESS_OBJECTIVE o
            ON r.OBJECTIVE_ID = o.OBJECTIVE_ID
        LEFT JOIN REQUIREMENT_SOURCE s
            ON r.REQUIREMENT_ID = s.REQUIREMENT_ID
        LEFT JOIN INTAKE_DOCUMENT d
            ON s.DOCUMENT_ID = d.DOCUMENT_ID
        LEFT JOIN VALIDATION_DECISION v
            ON r.REQUIREMENT_ID = v.REQUIREMENT_ID
            AND r.VERSION = v.REQUIREMENT_VERSION;

7. Fail baseline sign-off if an approved requirement has no source or validation record.
8. Use Horizon access history and audit views to report who accessed governed intake data.
9. Align retention with client policy and legal hold requirements.

## Coding Conventions

- Grant privileges to roles only, never directly to users.
- Do not copy restricted source text into logs, prompts, or notifications unless the program permits it.
- Test masking and row access policies with each role before release.
- Keep policy definitions in version control.
- Treat missing lineage as a release-blocking defect.