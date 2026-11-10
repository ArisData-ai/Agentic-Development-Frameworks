# Deployment Validation Agent

*Segment 5: Deploy & Release*

The Deployment Validation Agent checks that every Snowflake deployment landed correctly and works in the target environment. It catches failed or partial deployments within minutes of go-live and gives the Release Gatekeeper clear go/no-go evidence.

## How It Works

The agent runs a closed loop: **Sense, Act, Hand off**.

| Step | What happens | Snowflake components |
|------|--------------|----------------------|
| Sense | Detects completed deployments written to the deployment log by CI/CD | Streams, triggered Tasks |
| Act | Runs post-deployment checks on objects, permissions, data flows, and performance | Snowpark Python procedures, INFORMATION_SCHEMA, SHOW commands |
| Hand off | Scores the findings into a verdict, explains them, and notifies the Release Gatekeeper | Cortex AI, Cortex Agents, notification integrations |

The agent is read-only on target databases. It recommends a rollback but never performs one; only the Release Gatekeeper confirms or rolls back a release.

## Architecture

    CI/CD pipeline (GitHub Actions, Azure DevOps, Jenkins, schemachange, dbt, Terraform)
        |
        |  INSERT deployment row + manifest
        v
    DEPLOYMENT_LOG --> S_DEPLOYMENT_LOG (stream)
                            |
                            v
                  T_ENQUEUE_DEPLOYMENTS (triggered task) --> VALIDATION_QUEUE
                            |
                            v
                  T_RUN_VALIDATION --> RUN_DEPLOYMENT_VALIDATION
                                          |-- VALIDATE_OBJECTS
                                          |-- VALIDATE_PERMISSIONS
                                          |-- VALIDATE_DATA_FLOWS
                                          |-- VALIDATE_PERFORMANCE
                                          |-- BUILD_VERDICT_AND_ESCALATE
                                                  |
                    VALIDATION_RESULTS <----------+
                    VALIDATION_VERDICTS <---------+
                    RELEASE_GATEKEEPER_INBOX <----+--> Email / Slack / Teams alert
                                                  |
                    DEPLOYMENT_VALIDATION_AGENT (Cortex Agent) answers follow-up questions

## Folder Structure

    deployment-validation-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── deployment-validation-agent.md
        └── deployment-validation-triggers.md
        └── deployment-validation-object-permission-checks.md
        └── deployment-validation-data-flow-performance-checks.md
        └── deployment-validation-state-comparison-escalation.md

## Prerequisites

- A Snowflake account with Snowpark Python, Streams, Tasks, and Cortex AI enabled
- A role that can create databases, warehouses, roles, notification integrations, and agents (usually ACCOUNTADMIN for initial setup)
- A Cortex model available in your region, or cross-region inference enabled
- A CI/CD service user with key-pair authentication and INSERT access on `DEPLOYMENT_LOG`
- A deployment tool that can produce a manifest of expected objects, grants, data flows, and performance checks

## Setup

Run the steps in this order. Each step points to the skill file that holds the code.

1. **Foundation:** Create `DEPLOY_VALIDATION_DB.CORE`, `VALIDATION_WH`, `DEPLOY_VALIDATOR_ROLE`, and the control tables. (`SKILL`)
2. **Validator privileges:** Grant metadata visibility on each target database. (`deployment-validation-object-permission-checks`)
3. **Check procedures:** Create `VALIDATE_OBJECTS`, `VALIDATE_PERMISSIONS`, `VALIDATE_DATA_FLOWS`, `VALIDATE_PERFORMANCE`, and the `PERFORMANCE_BASELINES` table. (object-permission and data-flow-performance skills)
4. **Escalation:** Create `RELEASE_GATEKEEPER_INBOX`, `VALIDATION_EMAIL_INT`, `BUILD_VERDICT_AND_ESCALATE`, the semantic view, and the Cortex Agent. (`deployment-validation-state-comparison-escalation`)
5. **Orchestrator:** Create `RUN_DEPLOYMENT_VALIDATION`. (`deployment-validation-agent`)
6. **Triggers:** Create the stream, queue processor, and tasks, then resume the child tasks before the root task. (`deployment-validation-triggers`)
7. **CI/CD:** Add the write-back step as the last step of every deployment job. (`deployment-validation-triggers`)

## Deployment Manifest

The CI/CD pipeline writes a JSON manifest into `DEPLOYMENT_LOG.MANIFEST`. Every check reads its expected state from it.

    {
      "objects":     [{"type": "TABLE", "name": "SALES_DB.CORE.ORDERS", "columns": [{"name": "ORDER_ID", "type": "NUMBER"}]}],
      "grants":      [{"role": "ANALYST_ROLE", "privilege": "SELECT", "on": "VIEW", "name": "SALES_DB.MART.V_DAILY_REVENUE"}],
      "data_flows":  [{"type": "FRESHNESS", "name": "SALES_DB.CORE.ORDERS", "ts_column": "ORDER_TS", "max_minutes": 60}],
      "performance": [{"name": "daily_revenue_smoke", "sql": "SELECT COUNT(*) FROM SALES_DB.MART.V_DAILY_REVENUE", "baseline_ms": 1200, "tolerance_pct": 50}]
    }

Supported `objects` types: TABLE, VIEW, DYNAMIC_TABLE, PROCEDURE, FUNCTION, TASK, STREAM.
Supported `data_flows` types: TASK, STREAM, DYNAMIC_TABLE, PIPE, FRESHNESS.

Generate the manifest from your deployment tool's change set (schemachange scripts, dbt manifest.json, Terraform plan) rather than writing it by hand.

## Usage

**Automatic:** Insert a `COMPLETED` row into `DEPLOYMENT_LOG`. Validation starts as soon as the stream has data.

**Manual run:**

    CALL DEPLOY_VALIDATION_DB.CORE.RUN_DEPLOYMENT_VALIDATION('<deployment_id>');

**Re-queue a deployment:**

    INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_QUEUE (DEPLOYMENT_ID) VALUES ('<deployment_id>');
    EXECUTE TASK DEPLOY_VALIDATION_DB.CORE.T_SWEEP_STUCK_VALIDATIONS;

**Review the latest verdicts:**

    SELECT RELEASE_ID, ENVIRONMENT, VERDICT, CRITICAL_FAILS, HIGH_FAILS, WARNINGS, SUMMARY, VALIDATION_SECS
    FROM DEPLOY_VALIDATION_DB.CORE.VALIDATION_VERDICTS
    ORDER BY CREATED_AT DESC
    LIMIT 20;

**Record the Gatekeeper decision:**

    UPDATE DEPLOY_VALIDATION_DB.CORE.RELEASE_GATEKEEPER_INBOX
    SET DECISION = 'CONFIRMED', DECIDED_BY = CURRENT_USER(), DECIDED_AT = CURRENT_TIMESTAMP()
    WHERE RUN_ID = '<run_id>' AND DECISION IS NULL;

**Ask the agent:** Use `DEPLOYMENT_VALIDATION_AGENT` in Snowflake Intelligence or the Cortex Agents API, for example "Why did release R-2026.11 fail in PROD?"

## Verdict Rules

| Condition | Verdict | Recommendation |
|-----------|---------|----------------|
| Any CRITICAL or HIGH failure, or any ERROR | NO_GO | ROLLBACK |
| Only warnings or MEDIUM/LOW failures | GO_WITH_WARNINGS | REVIEW_WARNINGS |
| All checks pass | GO | CONFIRM_RELEASE |

The verdict comes from fixed SQL rules. Cortex AI only writes the plain-language summary; it never decides the verdict.

## Severity Defaults

| Check | Severity |
|-------|----------|
| Missing table, view, or dynamic table | CRITICAL |
| Grants to PUBLIC on release objects | CRITICAL |
| Stale data on consumer-facing tables | CRITICAL |
| Column mismatch, task state, missing grant, failed task, stale stream, failed refresh, paused pipe | HIGH |
| Query runtime regression | MEDIUM |

## Operations

- **SLA:** Under 10 minutes from `COMPLETED_AT` to verdict, tracked in `VALIDATION_VERDICTS.VALIDATION_SECS`.
- **Second pass:** Schedule another validation 15 to 30 minutes after go-live for flows on longer schedules.
- **Baselines:** Refresh `PERFORMANCE_BASELINES` only from runs the Gatekeeper approved, using a weighted average.
- **Cost:** `VALIDATION_WH` is XSMALL with 60-second auto-suspend; Cortex is called only when the verdict is not GO.

## Troubleshooting

| Symptom | Check |
|---------|-------|
| Validation never started | Status is `COMPLETED`; tasks are resumed; `TASK_HISTORY` for `T_ENQUEUE_DEPLOYMENTS` |
| Queue items stay unprocessed | `T_SWEEP_STUCK_VALIDATIONS` is running; errors in `T_RUN_VALIDATION` history |
| Checks return ERROR | `DETAILS` column in `VALIDATION_RESULTS`; validator role privileges on the target database |
| Data flow returns WARN "no history yet" | Expected right after deployment; wait for the second pass |
| No email received | Recipient is in `ALLOWED_RECIPIENTS`; the integration is enabled |
| Cortex summary fails | Model is available in the region, or cross-region inference is enabled |

## Security

- The validator role holds only USAGE, MONITOR, and REFERENCES on target databases; it holds no SELECT on sensitive data and no MANAGE GRANTS without security approval.
- The agent only inserts rows; only the Release Gatekeeper updates decision columns.
- CI/CD service users authenticate with key pairs; pipeline secrets never appear in SQL.
- Cortex Agent instructions forbid changing verdicts or performing rollbacks.

## Who Benefits

- **Release owners:** Fast, clear go/no-go evidence.
- **Operations teams:** Fewer post-release incidents.
- **Business users:** Fewer disruptions to reports and data products.

## Ideal Customer Profile

Teams with business-critical data products, where a failed release has immediate revenue or reporting impact.