---
name: pipeline-openflow-ingestion
description: This skill defines how the Pipeline Agent assembles governed ingestion flows in Snowflake Openflow, converting approved source mappings into reusable connector configurations, raw landing objects, and ingestion validation evidence. Use this skill for any task involving Openflow, source connectors, batch or change data capture ingestion, landing data from SAP, Oracle, Workday, databases, files, or APIs, configuring raw tables, parameterizing flows, or troubleshooting ingestion definitions.
---

# Pipeline Openflow Ingestion

## Overview
This skill covers the source-to-raw portion of an ELT pipeline. It uses Openflow to move approved source data into Snowflake while preserving source fidelity and load metadata. Business transformations belong in later pipeline layers.

## When to Use
- A build manifest includes a new source or source object.
- An approved mapping changes ingested fields or objects.
- A legacy ingestion job is being replaced with Openflow.
- An ingestion definition needs revision after review.

## Instructions
1. **Read the manifest.** Confirm the source objects, fields, load pattern, sensitivity, and required freshness.
2. **Select the ingestion path.** Use a suitable Openflow connector when one is available for the source and required load pattern. Use a custom flow only when no connector meets the requirement, and record the reason.
3. **Configure access.** Use approved service identities, secret references, and least-privilege roles. Stop and flag the issue if access has not been approved.
4. **Define raw landing objects.** Keep source-aligned names and structures. Apply only technical changes required for loading.
5. **Add load metadata.** Capture the source system, load time, and batch or change identifier.
6. **Parameterize the flow.** Externalize environment-specific values so that one definition can be promoted across environments.
7. **Validate in non-production.** Confirm connectivity, target object creation, row movement, and expected incremental behavior. Include the results in the review package.

## Coding Conventions
- Keep sensitive values out of flow definitions, source control, and logs.
- Keep raw data separate from curated data.
- Do not apply business rules during ingestion.
- Use a consistent raw naming pattern:

        <RAW_DATABASE>.<SOURCE_SYSTEM>.<SOURCE_OBJECT>

- Add standard metadata columns when the selected ingestion path does not supply equivalents:

        CREATE TABLE IF NOT EXISTS RAW_DB.SAP.SALES_ORDERS (
            ORDER_ID        STRING,
            CUSTOMER_ID     STRING,
            ORDER_DATE      DATE,
            NET_VALUE       NUMBER(18,2),
            _SOURCE_SYSTEM  STRING,
            _LOADED_AT      TIMESTAMP_NTZ,
            _BATCH_ID       STRING
        );