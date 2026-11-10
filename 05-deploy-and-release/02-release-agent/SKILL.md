---
name: release-agent
description: This skill defines the end-to-end Release Agent workflow that senses test-passed changes, assembles a release package with scope, dependency checks, release notes, and an audit trail, then submits it to the Release Gatekeeper, orchestrated through Cortex Agents, Snowflake tasks and streams, Snowflake Horizon, and Cortex Search. Use this skill for any task involving release preparation, release package orchestration, go-live readiness, change-management automation, building or configuring a Cortex Agent for releases, or wiring the release stage between CI/CD and deployment validation in a Snowflake delivery pipeline.
---
# Release Agent

## Overview
The Release Agent operates in Segment 5 (Deploy and Release) of the AI-driven delivery pipeline. It turns tested changes into a complete, traceable release package so go-live decisions rest on evidence rather than manual assembly. It follows a Sense, Act, Hand Off loop:

- Sense: detect changes that passed testing and are queued for release.
- Act: assemble scope, check dependencies, draft release notes, record an audit log.
- Hand Off: submit the package to the Release Gatekeeper, who approves, defers, or modifies it.

Snowflake components:
- Cortex Agents: coordinate the release preparation steps and tool calls.
- Snowflake Horizon: retains audit, tagging, and lineage data.
- Cortex Search: makes past releases and notes searchable.
- Streams and Tasks: event-driven triggering of the release cycle.

Target customers are organizations with frequent releases and strict change-management rules where manual release prep is slow and error-prone. Beneficiaries are release managers (no manual packaging), business stakeholders (clear visibility into what changes and when), and compliance teams (complete release records).

## When to Use
- Standing up the Release Agent in a new Snowflake account or client environment.
- Defining the release lifecycle, state model, and orchestration schedule.
- Creating or updating the Cortex Agent specification for release preparation.
- Integrating upstream Testing and Reconciliation agents with downstream CI/CD and Deployment Validation agents.
- Diagnosing a stalled or failed release cycle.

## Instructions

1. Provision the release control plane.

    CREATE DATABASE IF NOT EXISTS RELEASE_DB;
    CREATE SCHEMA IF NOT EXISTS RELEASE_DB.CORE;
    CREATE SCHEMA IF NOT EXISTS RELEASE_DB.AUDIT;
    CREATE SCHEMA IF NOT EXISTS RELEASE_DB.SEARCH;

    CREATE ROLE IF NOT EXISTS RELEASE_AGENT_ROLE;
    CREATE ROLE IF NOT EXISTS RELEASE_GATEKEEPER_ROLE;
    CREATE ROLE IF NOT EXISTS RELEASE_AUDITOR_ROLE;

    CREATE WAREHOUSE IF NOT EXISTS RELEASE_WH
        WAREHOUSE_SIZE = 'XSMALL'
        AUTO_SUSPEND = 60
        AUTO_RESUME = TRUE
        INITIALLY_SUSPENDED = TRUE;

2. Create the core release tables.

    CREATE TABLE IF NOT EXISTS RELEASE_DB.CORE.RELEASE_CANDIDATES (
        CHANGE_ID        STRING NOT NULL,
        SOURCE_REPO      STRING,
        COMMIT_SHA       STRING,
        TICKET_ID        STRING,
        OBJECT_FQN       STRING,
        CHANGE_TYPE      STRING,          -- CREATE | ALTER | DROP | DATA
        TEST_RUN_ID      STRING,
        TEST_STATUS      STRING,          -- PASSED | FAILED
        QUEUED_AT        TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        RELEASE_ID       STRING,
        CONSTRAINT PK_CANDIDATE PRIMARY KEY (CHANGE_ID)
    );

    CREATE TABLE IF NOT EXISTS RELEASE_DB.CORE.RELEASE_PACKAGES (
        RELEASE_ID       STRING NOT NULL,
        RELEASE_VERSION  STRING,
        TARGET_ENV       STRING,
        STATUS           STRING,          -- DRAFT | READY | SUBMITTED | APPROVED | DEFERRED | MODIFY_REQUESTED | RELEASED
        MANIFEST         VARIANT,
        DEPENDENCY_REPORT VARIANT,
        RELEASE_NOTES    VARIANT,
        PLANNED_WINDOW   STRING,
        CREATED_BY       STRING DEFAULT CURRENT_USER(),
        CREATED_AT       TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        UPDATED_AT       TIMESTAMP_LTZ,
        CONSTRAINT PK_RELEASE PRIMARY KEY (RELEASE_ID)
    );

3. Enforce the release state machine. Allowed transitions:

    DRAFT            -> READY | DRAFT
    READY            -> SUBMITTED
    SUBMITTED        -> APPROVED | DEFERRED | MODIFY_REQUESTED
    MODIFY_REQUESTED -> DRAFT
    DEFERRED         -> READY
    APPROVED         -> RELEASED

    Every transition writes one row to RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG (see release-audit-trail).

4. Trigger the cycle from test results with a stream and task.

    CREATE OR REPLACE STREAM RELEASE_DB.CORE.CANDIDATES_STREAM
        ON TABLE RELEASE_DB.CORE.RELEASE_CANDIDATES
        APPEND_ONLY = TRUE;

    CREATE OR REPLACE TASK RELEASE_DB.CORE.RUN_RELEASE_CYCLE
        WAREHOUSE = RELEASE_WH
        SCHEDULE = '15 MINUTE'
        WHEN SYSTEM$STREAM_HAS_DATA('RELEASE_DB.CORE.CANDIDATES_STREAM')
    AS
        CALL RELEASE_DB.CORE.SP_PREPARE_RELEASE('PROD');

    ALTER TASK RELEASE_DB.CORE.RUN_RELEASE_CYCLE RESUME;

5. Implement SP_PREPARE_RELEASE as an orchestration procedure that calls, in order:
    - SP_ASSEMBLE_SCOPE (release-scope-assembly)
    - SP_CHECK_DEPENDENCIES (release-scope-assembly)
    - SP_DRAFT_RELEASE_NOTES (release-notes-generation)
    - SP_LOG_RELEASE_EVENT (release-audit-trail)
    - SP_SUBMIT_TO_GATEKEEPER (release-gatekeeper-handoff)

    Stop the pipeline and set STATUS = 'DRAFT' with a blocking reason if any dependency check fails.

6. Define the Cortex Agent for interactive and orchestrated release prep.

    CREATE OR REPLACE AGENT RELEASE_DB.CORE.RELEASE_AGENT
        COMMENT = 'Prepares release packages, notes, and audit trails'
        FROM SPECIFICATION
        $$
        models:
            orchestration: auto
        instructions:
            response: "Answer as a release coordinator. Cite RELEASE_ID and CHANGE_ID values."
            orchestration: "Assemble scope first, then check dependencies, then draft notes, then log and submit. Never submit a package with unresolved blocking dependencies."
        tools:
            - tool_spec:
                type: cortex_search
                name: release_history
            - tool_spec:
                type: generic
                name: prepare_release
                description: "Runs SP_PREPARE_RELEASE for a target environment"
        tool_resources:
            release_history:
                name: RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC
                max_results: 5
            prepare_release:
                type: procedure
                identifier: RELEASE_DB.CORE.SP_PREPARE_RELEASE
                execution_environment:
                    type: warehouse
                    warehouse: RELEASE_WH
        $$;

7. Grant least-privilege access.

    GRANT USAGE ON DATABASE RELEASE_DB TO ROLE RELEASE_AGENT_ROLE;
    GRANT USAGE ON ALL SCHEMAS IN DATABASE RELEASE_DB TO ROLE RELEASE_AGENT_ROLE;
    GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA RELEASE_DB.CORE TO ROLE RELEASE_AGENT_ROLE;
    GRANT INSERT ON ALL TABLES IN SCHEMA RELEASE_DB.AUDIT TO ROLE RELEASE_AGENT_ROLE;
    GRANT SELECT ON ALL TABLES IN SCHEMA RELEASE_DB.AUDIT TO ROLE RELEASE_AUDITOR_ROLE;
    GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE RELEASE_AGENT_ROLE;
    GRANT DATABASE ROLE SNOWFLAKE.CORTEX_USER TO ROLE RELEASE_AGENT_ROLE;

8. Validate end to end: insert a PASSED candidate, execute the task manually, and confirm a SUBMITTED package with notes, dependency report, and audit rows.

    EXECUTE TASK RELEASE_DB.CORE.RUN_RELEASE_CYCLE;
    SELECT RELEASE_ID, STATUS, MANIFEST:change_count FROM RELEASE_DB.CORE.RELEASE_PACKAGES ORDER BY CREATED_AT DESC LIMIT 1;

Best practices:
- Never include a change whose TEST_STATUS is not PASSED.
- Keep the agent stateless; all release state lives in RELEASE_PACKAGES.
- The agent prepares and recommends; only the Release Gatekeeper approves.
- Make every procedure idempotent so task retries do not duplicate packages.

Output expectations: a RELEASE_PACKAGES row in SUBMITTED status with a JSON manifest, dependency report, audience-specific release notes, a complete audit trail, and a gatekeeper notification.

## Coding Conventions
- Object names in UPPER_SNAKE_CASE; procedures prefixed SP_, streams suffixed _STREAM, search services suffixed _SVC.
- RELEASE_ID format: REL-<ENV>-<YYYYMMDD>-<SEQ>, for example REL-PROD-20261110-001.
- RELEASE_VERSION follows semantic versioning (MAJOR.MINOR.PATCH).
- Use fully qualified names in all procedures and task bodies.
- Store structured artifacts as VARIANT JSON with snake_case keys.
- Use CREATE OR REPLACE only for stateless objects; use CREATE IF NOT EXISTS for tables.
- Run all agent work under RELEASE_AGENT_ROLE and RELEASE_WH; never use ACCOUNTADMIN.

## References
- references/release-scope-assembly.md: detecting test-passed changes, building the release manifest, dependency and conflict checks, deployment ordering.
- references/release-notes-generation.md: drafting audience-specific release notes and changelogs with Cortex AI_COMPLETE.
- references/release-audit-trail.md: append-only audit log, Horizon tagging, lineage snapshots, retention, compliance queries.
- references/release-gatekeeper-handoff.md: package submission, approve, defer, or modify decision handling, notifications, and handoff to CI/CD.
- references/release-history-search.md: Cortex Search service over past releases for lookups and agent retrieval.