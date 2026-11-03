---
name: discovery-source-metadata-extraction
description: This skill defines how a Discovery Agent extracts metadata from source systems, stores versioned snapshots, and classifies new, changed, and removed assets through Openflow, Snowflake tables, and Cortex Agents. Use this skill for any task involving source connectivity, schema scanning, job or interface extraction, metadata snapshots, or change detection, including Openflow connector setup and scans of Teradata, Oracle, SQL Server, SAP, Informatica, SSIS, Airflow, Control-M, files, and APIs.
---

# Discovery Source Metadata Extraction

## Overview

This skill covers the Sense phase of the Discovery Agent. It captures metadata about systems, schemas, jobs, and interfaces, then compares each sweep with the previous snapshot to find changes.

## When to Use

- Connect a new source system to discovery.
- Configure metadata extraction through Openflow.
- Capture schemas, tables, views, procedures, jobs, or interfaces.
- Detect new, changed, or removed assets between sweeps.
- Investigate missing or incomplete source metadata.

## Instructions

1. Confirm that the source owner has approved read-only metadata access.
2. Use Openflow where a suitable connection is available. For unsupported sources, use an approved export and record the extraction method.
3. Capture metadata only. Do not extract business records for discovery.
4. Store each asset as an immutable snapshot record with a stable qualified name.
5. Exclude volatile fields, such as row counts or last-read timestamps, from the change hash unless they are part of the discovery requirement.
6. Compare the snapshot only with the latest complete sweep for that source.
7. Mark inaccessible assets as `UNVERIFIED`. Do not classify them as removed.

Create the snapshot table:

    CREATE TABLE IF NOT EXISTS DISCOVERY_DB.RAW.SOURCE_METADATA_SNAPSHOT (
        SWEEP_ID                STRING NOT NULL,
        SOURCE_SYSTEM_ID        STRING NOT NULL,
        ASSET_TYPE              STRING NOT NULL,
        ASSET_QUALIFIED_NAME    STRING NOT NULL,
        ATTRIBUTES              VARIANT,
        ATTRIBUTES_HASH         STRING NOT NULL,
        EXTRACTION_METHOD       STRING NOT NULL,
        CAPTURED_AT             TIMESTAMP_NTZ
            DEFAULT CURRENT_TIMESTAMP()
    );

Classify changes between two sweeps:

    SET CURRENT_SWEEP_ID = '<current_sweep_id>';
    SET PREVIOUS_SWEEP_ID = '<previous_complete_sweep_id>';

    WITH CURRENT_SNAPSHOT AS (
        SELECT *
        FROM DISCOVERY_DB.RAW.SOURCE_METADATA_SNAPSHOT
        WHERE SWEEP_ID = $CURRENT_SWEEP_ID
    ),
    PREVIOUS_SNAPSHOT AS (
        SELECT *
        FROM DISCOVERY_DB.RAW.SOURCE_METADATA_SNAPSHOT
        WHERE SWEEP_ID = $PREVIOUS_SWEEP_ID
    )
    SELECT
        COALESCE(C.SOURCE_SYSTEM_ID, P.SOURCE_SYSTEM_ID)
            AS SOURCE_SYSTEM_ID,
        COALESCE(C.ASSET_TYPE, P.ASSET_TYPE)
            AS ASSET_TYPE,
        COALESCE(C.ASSET_QUALIFIED_NAME, P.ASSET_QUALIFIED_NAME)
            AS ASSET_QUALIFIED_NAME,
        CASE
            WHEN P.ASSET_QUALIFIED_NAME IS NULL THEN 'NEW'
            WHEN C.ASSET_QUALIFIED_NAME IS NULL THEN 'REMOVED'
            ELSE 'CHANGED'
        END AS CHANGE_TYPE
    FROM CURRENT_SNAPSHOT C
    FULL OUTER JOIN PREVIOUS_SNAPSHOT P
        ON C.SOURCE_SYSTEM_ID = P.SOURCE_SYSTEM_ID
        AND C.ASSET_TYPE = P.ASSET_TYPE
        AND C.ASSET_QUALIFIED_NAME = P.ASSET_QUALIFIED_NAME
    WHERE C.ATTRIBUTES_HASH IS DISTINCT FROM P.ATTRIBUTES_HASH;

## Coding Conventions

- Keep snapshots append-only.
- Normalize names consistently before hashing.
- Serialize attributes in a stable key order before hashing.
- Use `VARIANT` for source-specific attributes.
- Record the extraction method for every asset.
- Keep credentials in approved secret objects or Openflow secret management.