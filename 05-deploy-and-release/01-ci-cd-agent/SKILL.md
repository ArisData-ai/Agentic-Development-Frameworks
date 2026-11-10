---
name: ci-cd-agent
description: This skill defines the end-to-end CI/CD Agent workflow that detects merged changes in a Snowflake Git repository, triggers builds and automated checks, prepares promotion between environments, and presents promotion readiness to a Release Gatekeeper, coordinated through Cortex Agents, Snowpark, Streams, and Tasks. Use this skill for any task involving designing, implementing, or operating a Snowflake CI/CD agent, including standardizing how data changes are packaged and promoted from DEV to QA to PROD, replacing manual deployments, wiring GitHub Actions, GitLab CI, or Azure DevOps to Snowflake CLI, setting up a control schema for pipeline runs, and reducing environment drift.
---

# CI/CD Agent

## Overview

The CI/CD Agent belongs to Segment 5 (Deploy and Release) of the agentic data project pipeline. It runs build and promotion workflows so approved changes move through environments the same way every time. The agent works in three stages:

- Senses: detects merged commits on tracked branches and new build artifacts on stage.
- Acts: triggers builds, runs automated checks, and prepares promotion to the next environment. It halts on the first blocking failure.
- Hands off: publishes a promotion readiness report to the Release Gatekeeper, a human or role-bound approver who approves or blocks each move.

The agent never promotes to a protected environment (QA or PROD by default) without a recorded Gatekeeper decision.

Benefits:

- DevOps engineers get one repeatable path to production.
- Developers get fast pipeline feedback for every merge.
- Platform owners see less drift between DEV, QA, and PROD.

Target customers are teams that still promote data changes by hand, where inconsistent deployments cause outages and environment drift.

## When to Use

- Building a new CI/CD path for Snowflake objects, Snowpark code, Streams, Tasks, or dynamic tables.
- Replacing manual worksheet-based deployments with a governed, auditable promotion flow.
- Adding agent-driven orchestration and failure triage on top of an existing CI runner.
- Standing up the control schema that every sibling skill reads and writes.

## Instructions

1. Provision the CI/CD control plane once per account.

        CREATE DATABASE IF NOT EXISTS CICD_DB;
        CREATE SCHEMA IF NOT EXISTS CICD_DB.CONTROL;
        CREATE SCHEMA IF NOT EXISTS CICD_DB.SRC;
        CREATE SCHEMA IF NOT EXISTS CICD_DB.ARTIFACTS;
        CREATE SCHEMA IF NOT EXISTS CICD_DB.ORCH;
        CREATE SCHEMA IF NOT EXISTS CICD_DB.SECRETS;

        CREATE WAREHOUSE IF NOT EXISTS CICD_WH
            WAREHOUSE_SIZE = XSMALL
            AUTO_SUSPEND = 60
            AUTO_RESUME = TRUE
            INITIALLY_SUSPENDED = TRUE;

        CREATE TABLE IF NOT EXISTS CICD_DB.CONTROL.PIPELINE_RUNS (
            RUN_ID          STRING DEFAULT UUID_STRING(),
            BRANCH          STRING,
            COMMIT_HASH     STRING,
            TARGET_ENV      STRING,
            STATUS          STRING,   -- DETECTED | BUILDING | CHECKING | READY | APPROVED | REJECTED | PROMOTED | FAILED
            FAILURE_STAGE   STRING,
            FAILURE_DETAIL  VARIANT,
            CREATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
            UPDATED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS CICD_DB.CONTROL.CHECK_RESULTS (
            RUN_ID      STRING,
            CHECK_NAME  STRING,
            CHECK_TYPE  STRING,       -- BUILD | UNIT | DATA_QUALITY | DRIFT | POLICY
            SEVERITY    STRING,       -- BLOCKING | WARNING
            PASSED      BOOLEAN,
            DETAIL      VARIANT,
            CHECKED_AT  TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS CICD_DB.CONTROL.PROMOTION_REQUESTS (
            REQUEST_ID      STRING DEFAULT UUID_STRING(),
            RUN_ID          STRING,
            SOURCE_ENV      STRING,
            TARGET_ENV      STRING,
            READINESS       VARIANT,
            DECISION        STRING,   -- PENDING | APPROVED | REJECTED
            DECIDED_BY      STRING,
            DECISION_NOTE   STRING,
            REQUESTED_AT    TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
            DECIDED_AT      TIMESTAMP_LTZ
        );

2. Create the role model and keep separation of duties.

        CREATE ROLE IF NOT EXISTS CICD_DEPLOYER;        -- agent and CI runner
        CREATE ROLE IF NOT EXISTS RELEASE_GATEKEEPER;   -- human approvers only
        CREATE ROLE IF NOT EXISTS CICD_OBSERVER;        -- read-only audit

        GRANT USAGE ON DATABASE CICD_DB TO ROLE CICD_DEPLOYER;
        GRANT USAGE ON ALL SCHEMAS IN DATABASE CICD_DB TO ROLE CICD_DEPLOYER;
        GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA CICD_DB.CONTROL TO ROLE CICD_DEPLOYER;
        GRANT USAGE ON WAREHOUSE CICD_WH TO ROLE CICD_DEPLOYER;
        GRANT SELECT ON ALL TABLES IN SCHEMA CICD_DB.CONTROL TO ROLE CICD_OBSERVER;

3. Run the agent lifecycle in this exact order, with one sibling skill per stage:
    - Detect: record a DETECTED run for each new commit (ci-cd-change-detection).
    - Build: package Snowpark code and render SQL for the target environment (ci-cd-build-and-checks).
    - Check: run blocking and warning checks, and halt on any blocking failure (ci-cd-build-and-checks).
    - Prepare: deploy to the non-protected environment, then compute drift and readiness for the next one (ci-cd-environment-promotion).
    - Hand off: create a PENDING promotion request and notify the Gatekeeper (ci-cd-release-gatekeeper-handoff).
    - Promote: run only after an APPROVED decision, deploying the identical commit hash (ci-cd-environment-promotion).
    - Orchestrate and recover: run the task graph, finalizer, retries, and Cortex Agent triage (ci-cd-orchestration-and-failure-handling).

4. Enforce these guardrails in every stage:
    - Build once, promote the same artifact. Never rebuild for QA or PROD from a different commit.
    - Every state transition updates PIPELINE_RUNS.STATUS and UPDATED_AT.
    - A blocking failure sets STATUS = 'FAILED', records FAILURE_STAGE, and stops downstream tasks.
    - Protected environments require a PROMOTION_REQUESTS row with DECISION = 'APPROVED'.
    - Secrets live only in Snowflake SECRET objects or the CI runner's secret store, never in the repository.

## Coding Conventions

- Environment databases follow <DOMAIN>_<ENV>_DB, for example SALES_DEV_DB, SALES_QA_DB, and SALES_PROD_DB.
- Control objects live in CICD_DB and use upper snake case. Procedures use the SP_ prefix and tasks use the T_ prefix.
- Prefer CREATE OR ALTER for tables, schemas, tasks, and views so deployments stay idempotent. Use CREATE OR REPLACE only for stateless objects.
- Parameterize environment-specific values through Jinja variables in EXECUTE IMMEDIATE FROM, or through Snowflake CLI config. Do not hardcode them.
- All procedures run with EXECUTE AS OWNER and are owned by CICD_DEPLOYER.
- Use 4-space indentation in SQL, Python, and YAML.

## References

- references/ci-cd-change-detection.md: Git repository integration, commit polling, artifact stage streams, and DETECTED run creation.
- references/ci-cd-build-and-checks.md: Snowpark packaging, templated SQL rendering, unit, data quality, and policy checks, and halt-on-failure logic.
- references/ci-cd-environment-promotion.md: DEV to QA to PROD promotion, immutable artifact deployment, zero-copy clone test data, and drift detection.
- references/ci-cd-release-gatekeeper-handoff.md: readiness report generation, approval and rejection procedures, notifications, and separation of duties.
- references/ci-cd-orchestration-and-failure-handling.md: Task graph design, finalizer tasks, retries, error notifications, and Cortex Agent coordination and failure triage.