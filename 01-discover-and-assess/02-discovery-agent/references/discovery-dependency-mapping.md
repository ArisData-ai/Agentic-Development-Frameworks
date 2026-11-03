---
name: discovery-dependency-mapping
description: This skill defines how a Discovery Agent traces upstream and downstream relationships, calculates impact paths, and flags undocumented or orphaned components through Snowflake tables, Snowflake Horizon lineage, and Cortex Agents. Use this skill for any task involving dependency maps, lineage tracing, impact analysis, hidden coupling, cycle detection, or orphan detection, including dependencies across SQL code, stored procedures, Informatica mappings, SSIS packages, Airflow DAGs, Control-M jobs, interfaces, and reports.
---

# Discovery Dependency Mapping

## Overview

This skill covers the dependency portion of the Act phase. It connects cataloged assets into a dependency graph so teams can identify hidden coupling, migration risk, and redundant components.

## When to Use

- Trace upstream or downstream dependencies.
- Identify the blast radius of a system or asset change.
- Build a migration dependency map.
- Detect circular dependencies.
- Find undocumented or orphaned components.

## Instructions

1. Collect dependency evidence from source metadata, code definitions, job configurations, interface records, and Horizon lineage where available.
2. Link only assets that resolve to catalog records.
3. Store unresolved references as exceptions. Do not create guessed relationships.
4. Assign each dependency an evidence type and confidence level.
5. Traverse dependencies with cycle protection and a depth limit.
6. Flag an asset as a potential orphan only when it has no dependencies and no recent usage evidence.
7. Treat orphan status as a finding. Retirement requires human review.

Create the dependency table:

    CREATE TABLE IF NOT EXISTS DISCOVERY_DB.LINEAGE.ASSET_DEPENDENCY (
        UPSTREAM_ASSET_ID       STRING NOT NULL,
        DOWNSTREAM_ASSET_ID     STRING NOT NULL,
        DEPENDENCY_TYPE         STRING NOT NULL,
        EVIDENCE_TYPE           STRING NOT NULL,
        CONFIDENCE_LEVEL        STRING NOT NULL,
        EVIDENCE                VARIANT,
        LAST_SEEN_SWEEP_ID      STRING NOT NULL,
        IS_CURRENT              BOOLEAN DEFAULT TRUE,
        CONSTRAINT PK_ASSET_DEPENDENCY PRIMARY KEY (
            UPSTREAM_ASSET_ID,
            DOWNSTREAM_ASSET_ID,
            DEPENDENCY_TYPE
        )
    );

Trace downstream impact with cycle protection:

    SET ROOT_ASSET_ID = '<asset_id>';

    WITH RECURSIVE DOWNSTREAM_PATH AS (
        SELECT
            UPSTREAM_ASSET_ID,
            DOWNSTREAM_ASSET_ID,
            1 AS PATH_DEPTH,
            ARRAY_CONSTRUCT(UPSTREAM_ASSET_ID, DOWNSTREAM_ASSET_ID)
                AS ASSET_PATH
        FROM DISCOVERY_DB.LINEAGE.ASSET_DEPENDENCY
        WHERE UPSTREAM_ASSET_ID = $ROOT_ASSET_ID
            AND IS_CURRENT

        UNION ALL

        SELECT
            D.UPSTREAM_ASSET_ID,
            D.DOWNSTREAM_ASSET_ID,
            P.PATH_DEPTH + 1,
            ARRAY_APPEND(P.ASSET_PATH, D.DOWNSTREAM_ASSET_ID)
        FROM DISCOVERY_DB.LINEAGE.ASSET_DEPENDENCY D
        JOIN DOWNSTREAM_PATH P
            ON D.UPSTREAM_ASSET_ID = P.DOWNSTREAM_ASSET_ID
        WHERE D.IS_CURRENT
            AND P.PATH_DEPTH < 20
            AND NOT ARRAY_CONTAINS(
                D.DOWNSTREAM_ASSET_ID::VARIANT,
                P.ASSET_PATH
            )
    )
    SELECT *
    FROM DOWNSTREAM_PATH
    ORDER BY PATH_DEPTH, DOWNSTREAM_ASSET_ID;

Identify assets with no recorded dependencies:

    SELECT A.ASSET_ID, A.ASSET_TYPE, A.ASSET_QUALIFIED_NAME
    FROM DISCOVERY_DB.CATALOG.DISCOVERED_ASSET A
    WHERE A.LIFECYCLE_STATUS = 'ACTIVE'
        AND NOT EXISTS (
            SELECT 1
            FROM DISCOVERY_DB.LINEAGE.ASSET_DEPENDENCY D
            WHERE D.IS_CURRENT
                AND (
                    D.UPSTREAM_ASSET_ID = A.ASSET_ID
                    OR D.DOWNSTREAM_ASSET_ID = A.ASSET_ID
                )
        );

## Coding Conventions

- Keep the direction consistent: upstream assets feed downstream assets.
- Store dependency evidence for every edge.
- Separate confirmed, inferred, and unresolved relationships.
- Use cycle guards and depth limits in recursive queries.
- Keep unresolved references visible in the review package.
- Validate usage evidence before labeling an asset as orphaned.