---
name: discovery-asset-cataloging
description: This skill defines how a Discovery Agent standardizes extracted metadata into governed asset records with ownership, status, confidence, and evidence through Snowflake tables, Snowflake Horizon, and Cortex Agents. Use this skill for any task involving asset inventories, system catalogs, metadata standardization, duplicate resolution, ownership capture, or scope tagging, including catalogs for databases, schemas, tables, views, procedures, ETL jobs, reports, files, APIs, and legacy applications.
---

# Discovery Asset Cataloging

## Overview

This skill covers the catalog portion of the Act phase. It turns raw metadata into a consistent inventory that migration leads, architects, and operations teams can review.

## When to Use

- Create or update the discovery inventory.
- Standardize metadata from different source technologies.
- Resolve duplicate assets across sweeps or sources.
- Capture business and technical ownership.
- Apply scope and validation metadata through Snowflake Horizon.

## Instructions

1. Read new and changed assets from the snapshot process.
2. Map each asset to a standard type, such as `DATABASE`, `SCHEMA`, `TABLE`, `VIEW`, `PROCEDURE`, `JOB`, `INTERFACE`, `FILE`, or `REPORT`.
3. Generate a stable asset ID from the source system, asset type, and qualified name.
4. Preserve original source names and attributes.
5. Mark assets missing an owner, description, or usable definition as undocumented.
6. Do not infer ownership unless source metadata supports it. Use `UNKNOWN` until a person confirms the owner.
7. Assign a confidence level and retain the evidence behind it.
8. Retire assets only after the Lead Validator confirms the removal.

Create the catalog table:

    CREATE TABLE IF NOT EXISTS DISCOVERY_DB.CATALOG.DISCOVERED_ASSET (
        ASSET_ID                STRING NOT NULL,
        SOURCE_SYSTEM_ID        STRING NOT NULL,
        ASSET_TYPE              STRING NOT NULL,
        ASSET_QUALIFIED_NAME    STRING NOT NULL,
        BUSINESS_OWNER          STRING DEFAULT 'UNKNOWN',
        TECHNICAL_OWNER         STRING DEFAULT 'UNKNOWN',
        LIFECYCLE_STATUS        STRING DEFAULT 'ACTIVE',
        IS_DOCUMENTED           BOOLEAN DEFAULT FALSE,
        CONFIDENCE_LEVEL        STRING NOT NULL,
        EVIDENCE                VARIANT,
        FIRST_SEEN_SWEEP_ID     STRING NOT NULL,
        LAST_SEEN_SWEEP_ID      STRING NOT NULL,
        UPDATED_AT              TIMESTAMP_NTZ
            DEFAULT CURRENT_TIMESTAMP(),
        CONSTRAINT PK_DISCOVERED_ASSET PRIMARY KEY (ASSET_ID)
    );

Upsert discovered assets through a staged sweep result:

    MERGE INTO DISCOVERY_DB.CATALOG.DISCOVERED_ASSET T
    USING DISCOVERY_DB.CATALOG.V_ASSET_CANDIDATE S
        ON T.ASSET_ID = S.ASSET_ID
    WHEN MATCHED THEN UPDATE SET
        T.IS_DOCUMENTED = S.IS_DOCUMENTED,
        T.CONFIDENCE_LEVEL = S.CONFIDENCE_LEVEL,
        T.EVIDENCE = S.EVIDENCE,
        T.LAST_SEEN_SWEEP_ID = S.SWEEP_ID,
        T.UPDATED_AT = CURRENT_TIMESTAMP()
    WHEN NOT MATCHED THEN INSERT (
        ASSET_ID,
        SOURCE_SYSTEM_ID,
        ASSET_TYPE,
        ASSET_QUALIFIED_NAME,
        IS_DOCUMENTED,
        CONFIDENCE_LEVEL,
        EVIDENCE,
        FIRST_SEEN_SWEEP_ID,
        LAST_SEEN_SWEEP_ID
    )
    VALUES (
        S.ASSET_ID,
        S.SOURCE_SYSTEM_ID,
        S.ASSET_TYPE,
        S.ASSET_QUALIFIED_NAME,
        S.IS_DOCUMENTED,
        S.CONFIDENCE_LEVEL,
        S.EVIDENCE,
        S.SWEEP_ID,
        S.SWEEP_ID
    );

Apply a scope tag to Snowflake-hosted discovery objects:

    CREATE TAG IF NOT EXISTS DISCOVERY_DB.GOVERNANCE.DISCOVERY_SCOPE
        ALLOWED_VALUES 'IN_SCOPE', 'OUT_OF_SCOPE', 'PENDING_REVIEW';

    ALTER TABLE <database>.<schema>.<table>
        SET TAG DISCOVERY_DB.GOVERNANCE.DISCOVERY_SCOPE = 'PENDING_REVIEW';

## Coding Conventions

- Use deterministic asset IDs and upsert with `MERGE`.
- Use controlled values for asset type, lifecycle status, and confidence level.
- Preserve source-native names and attributes for traceability.
- Store evidence as structured data, not free-form text alone.
- Do not overwrite human-confirmed owners with extracted values.
- Apply Horizon tags only to objects that exist in Snowflake.