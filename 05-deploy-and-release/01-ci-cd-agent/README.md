# CI/CD Agent

**Segment 5: Deploy & Release** | Agentic Data Project Pipeline on Snowflake

The CI/CD Agent runs build and promotion workflows so approved data changes move through DEV, QA, and PROD the same way every time. It detects merged changes, builds and validates them once, prepares promotion, and hands readiness to a Release Gatekeeper. The Gatekeeper approves or blocks each move to a protected environment.

## Why This Agent

Teams that promote data changes by hand often get inconsistent deployments, outages, and drift between environments. This agent gives them:

- **One repeatable path to production** for DevOps engineers
- **Fast pipeline feedback** for developers on every merge
- **Less environment drift** for platform owners
- **Auditable approvals** for every protected promotion

## How It Works

    Merge to main / artifact upload
            |
            v
    [1] DETECT    -> new commit or artifact recorded as DETECTED
            |
            v
    [2] BUILD     -> Snowpark package + templated SQL, built once per commit
            |
            v
    [3] CHECK     -> unit, data quality, policy, drift checks (halts on BLOCKING failure)
            |
            v
    [4] PREPARE   -> deploy to DEV, compute readiness for next environment
            |
            v
    [5] HAND OFF  -> readiness report sent to Release Gatekeeper
            |
            v
    [6] PROMOTE   -> runs only after APPROVED, same commit hash, QA then PROD

A Snowflake Task graph runs the stages in dependency order. A finalizer task records every outcome. A Cortex Agent coordinates the stages and explains failures.

## Snowflake Components

| Component | Role in the Agent |
|---|---|
| Git Repository integration | Source of truth for code; commit-pinned deployments |
| Snowpark | Packaged code deployment (procedures, UDFs, UDTFs) |
| Streams | Detect new artifacts on stage directory tables |
| Tasks (task graph + finalizer) | Orchestrate build, check, and promotion steps |
| Cortex Agents | Coordinate stages, triage failures, answer status questions |
| Cortex AI_COMPLETE | Plain-language readiness summaries for approvers |
| Zero-copy cloning | QA test data refresh and pre-promotion rollback points |
| Notification integrations | Email and webhook alerts for approvals and failures |
| Snowflake CLI | CI runner integration (GitHub Actions, GitLab CI, Azure DevOps) |

## Skill Files

| Skill | Covers |
|---|---|
| `SKILL` | Primary skill: end-to-end workflow, control schema, roles, guardrails |
| `ci-cd-change-detection` | Git integration, commit polling, artifact stage streams |
| `ci-cd-build-and-checks` | Snowpark packaging, SQL rendering, automated checks, halt-on-failure |
| `ci-cd-environment-promotion` | DEV to QA to PROD promotion, cloning, drift detection, rollback |
| `ci-cd-release-gatekeeper-handoff` | Readiness reports, notifications, approve and reject decisions |
| `ci-cd-orchestration-and-failure-handling` | Task graph, retries, finalizer, Cortex Agent triage |

## Folder Structure

    ci-cd-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── ci-cd-change-detection.md
        └── ci-cd-build-and-checks.md
        └── ci-cd-environment-promotion.md
        └── ci-cd-release-gatekeeper-handoff.md
        └── ci-cd-orchestration-and-failure-handling.md

Expected layout of the data project repository the agent deploys:

    repo/
        snowflake.yml
        app/                    # Snowpark Python source
        tests/                  # pytest unit tests
        deploy/
            deploy.sql          # Jinja-templated entry point
            migrations/         # V001__init.sql, V002__...
        checks/
            data_quality.sql
            policy.sql

## Control Schema

All state lives in `CICD_DB.CONTROL`:

| Table | Purpose |
|---|---|
| `PIPELINE_RUNS` | One row per detected commit; lifecycle status and failure detail |
| `CHECK_RESULTS` | Every check outcome with severity and evidence |
| `PROMOTION_REQUESTS` | Readiness reports and Gatekeeper decisions |
| `ENVIRONMENTS` | Promotion order and protection flags |
| `ARTIFACT_EVENTS` | Artifact uploads consumed from the stage stream |

Run status lifecycle:

    DETECTED -> BUILDING -> CHECKING -> READY -> APPROVED -> PROMOTED
                                     \-> FAILED       \-> REJECTED

## Roles

| Role | Responsibility |
|---|---|
| `CICD_DEPLOYER` | Owned by the agent and CI runner; builds, checks, and executes promotions |
| `RELEASE_GATEKEEPER` | Human approvers only; approves or rejects, never deploys |
| `CICD_OBSERVER` | Read-only audit access to control tables |

## Quick Start

1. Create the control plane, warehouse, and roles (see `ci-cd-agent`).
2. Connect your Git provider and create the repository object (see `ci-cd-change-detection`).
3. Add the build, check, and promotion procedures (see `ci-cd-build-and-checks` and `ci-cd-environment-promotion`).
4. Set up the email notification integration and the decision procedure (see `ci-cd-release-gatekeeper-handoff`).
5. Create the task graph and the Cortex Agent, then enable the graph:

        SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('CICD_DB.ORCH.T_ROOT_DETECT');

6. Merge a change to `main` and check the run:

        SELECT RUN_ID, COMMIT_HASH, STATUS, FAILURE_STAGE, UPDATED_AT
        FROM CICD_DB.CONTROL.PIPELINE_RUNS
        ORDER BY CREATED_AT DESC
        LIMIT 10;

7. Approve a pending promotion as a Gatekeeper:

        USE ROLE RELEASE_GATEKEEPER;
        CALL CICD_DB.ORCH.SP_DECIDE_PROMOTION('<request_id>', 'APPROVED', 'Reviewed readiness report');

## Guardrails

- Build once per commit and promote the identical artifact through every environment.
- Never deploy to QA or PROD without a recorded `APPROVED` decision.
- The approver cannot be the author of the change.
- Any blocking check failure halts the run and marks it `FAILED`.
- Secrets live only in Snowflake `SECRET` objects or the CI runner's secret store.
- Cortex Agent tools are read and DEV-only; approval and promotion procedures are never exposed to the agent.

## Ideal Customer Profile

Teams that still promote data changes by hand, where inconsistent deployments cause outages and environment drift between DEV, QA, and PROD.