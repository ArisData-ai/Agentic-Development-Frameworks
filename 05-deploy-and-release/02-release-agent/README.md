# Release Agent

**Segment 5: Deploy and Release** | AI Agents for Whole Project Pipeline

The Release Agent prepares each release package, its release notes and its audit trail, so go-live decisions rest on complete evidence. It turns tested changes into predictable, traceable releases and sends each one to a human Release Gatekeeper for the final decision.

---

## How It Works

| Phase | Action |
|---|---|
| **Sense** | Detects changes that passed testing and are queued for release |
| **Act** | Assembles the release scope, checks dependencies, drafts release notes and records an audit log |
| **Hand Off** | Submits the package to the **Release Gatekeeper**, who approves, defers or modifies it |

```
Testing / Reconciliation Agents
            │
            ▼
   RELEASE_CANDIDATES ──► Stream ──► Task (15 min)
                                         │
                                         ▼
                              SP_PREPARE_RELEASE
          ┌──────────────┬───────────────┼───────────────┬──────────────┐
          ▼              ▼               ▼               ▼              ▼
   Assemble Scope   Check Deps    Draft Notes      Log Audit     Submit to Gatekeeper
                                  (AI_COMPLETE)    (Horizon)            │
                                                                        ▼
                                                     APPROVE / DEFER / MODIFY
                                                                        │
                                                                        ▼
                                                     CI/CD Agent ──► Deployment Validation Agent
```

---

## Who Benefits

- **Release managers:** no more building release packages by hand.
- **Business stakeholders:** a clear view of what is changing and when.
- **Compliance teams:** complete, queryable release records.

**Ideal customer profile:** organizations with frequent releases and strict change-management rules, where manual release prep is slow and error-prone.

---

## Snowflake Platform Components

| Component | Role |
|---|---|
| **Cortex Agents** | Coordinate the release preparation steps and tool calls |
| **Cortex AI_COMPLETE** | Drafts release notes for each audience from the manifest |
| **Cortex Search** | Makes past releases, notes and decisions searchable |
| **Snowflake Horizon** | Keeps audit, tagging and lineage data |
| **Streams and Tasks** | Start release cycles and process decisions when events arrive |
| **Notification Integrations** | Send Slack, Teams or email alerts to gatekeepers |
| **Git Repository** | Validates commits against the release branch |

---

## Skill Files

| Skill | Covers |
|---|---|
| `release-agent` | Primary skill: end-to-end workflow, control plane, state machine, Cortex Agent spec, grants |
| `release-scope-assembly` | Candidate selection, manifest build, dependency and conflict checks, deployment order |
| `release-notes-generation` | Release notes for business, engineering and compliance readers with AI_COMPLETE |
| `release-audit-trail` | Append-only audit log, lineage snapshots, Horizon tags, auditor evidence view |
| `release-gatekeeper-handoff` | Submission, approve/defer/modify handling, notifications, CI/CD handoff |
| `release-history-search` | Cortex Search service over release history, agent tool wiring |

## Folder Structure

    release-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── release-scope-assembly.md
        └── release-notes-generation.md
        └── release-audit-trail.md
        └── release-gatekeeper-handoff.md
        └── release-history-search.md

---

## Prerequisites

- A Snowflake account with Cortex AI, Cortex Search and Cortex Agents available in your region
- A role able to create databases, roles, warehouses, tasks and integrations (setup only)
- `SNOWFLAKE.CORTEX_USER` database role
- `IMPORTED PRIVILEGES` on the `SNOWFLAKE` database (for `ACCOUNT_USAGE` and lineage)
- Upstream Testing Agent writing PASSED or FAILED results to `RELEASE_CANDIDATES`
- Optional: a Slack or Teams webhook, plus a ServiceNow or Jira external access integration

---

## Quick Start

1. **Provision the control plane:** follow `release-agent` steps 1 to 2 (database, schemas, roles, warehouse, core tables).
2. **Create the audit layer:** follow `release-audit-trail` steps 1 to 5.
3. **Deploy the procedures** in this order:
   - `SP_LOG_RELEASE_EVENT`
   - `SP_ASSEMBLE_SCOPE`
   - `SP_CHECK_DEPENDENCIES`
   - `SP_DRAFT_RELEASE_NOTES`
   - `SP_SUBMIT_TO_GATEKEEPER`
   - `SP_APPLY_DECISIONS`
   - `SP_PREPARE_RELEASE`
4. **Seed the note templates:** follow `release-notes-generation` step 1.
5. **Create the search service:** follow `release-history-search` steps 1 to 2.
6. **Create the Cortex Agent:** follow `release-agent` step 6.
7. **Turn on automation:**

```sql
ALTER TASK RELEASE_DB.CORE.PROCESS_DECISIONS RESUME;
ALTER TASK RELEASE_DB.CORE.RUN_RELEASE_CYCLE RESUME;
```

8. **Run a smoke test:**

```sql
INSERT INTO RELEASE_DB.CORE.RELEASE_CANDIDATES
    (CHANGE_ID, SOURCE_REPO, COMMIT_SHA, TICKET_ID, OBJECT_FQN, CHANGE_TYPE, TEST_RUN_ID, TEST_STATUS)
VALUES
    ('CHG-DEMO1-a1b2c3d', 'app-repo', 'a1b2c3d', 'DEMO-1', 'ANALYTICS_DB.SALES.V_ORDERS', 'ALTER', 'TR-001', 'PASSED');

EXECUTE TASK RELEASE_DB.CORE.RUN_RELEASE_CYCLE;

SELECT RELEASE_ID, STATUS, MANIFEST:change_count, RELEASE_NOTES:BUSINESS
FROM RELEASE_DB.CORE.RELEASE_PACKAGES
ORDER BY CREATED_AT DESC
LIMIT 1;
```

The latest package should show `STATUS = 'SUBMITTED'`, with notes, a dependency report and audit events.

---

## Release Lifecycle

```
DRAFT ──► READY ──► SUBMITTED ──┬──► APPROVED ──► RELEASED
  ▲          ▲                  ├──► DEFERRED ──────┘ (returns to READY after DEFER_UNTIL)
  │          └──────────────────┘
  └──────────────────────────────────── MODIFY_REQUESTED
```

| Status | Set By | Meaning |
|---|---|---|
| `DRAFT` | Agent | Scope assembled; blocking items or notes are not yet valid |
| `READY` | Agent | Dependencies are clear and notes are validated |
| `SUBMITTED` | Agent | Sent to the Release Gatekeeper |
| `APPROVED` | Gatekeeper | Cleared for deployment |
| `DEFERRED` | Gatekeeper | Postponed until `DEFER_UNTIL` |
| `MODIFY_REQUESTED` | Gatekeeper | Scope or notes need changes |
| `RELEASED` | Deployment Validation Agent | Deployed and validated |

---

## Object Inventory

| Object | Type | Purpose |
|---|---|---|
| `RELEASE_DB.CORE.RELEASE_CANDIDATES` | Table | Tested changes waiting for release |
| `RELEASE_DB.CORE.RELEASE_PACKAGES` | Table | Release packages and their state |
| `RELEASE_DB.CORE.GATEKEEPER_DECISIONS` | Table | Approve, defer and modify decisions |
| `RELEASE_DB.CORE.NOTE_TEMPLATES` | Table | Versioned prompt templates for notes |
| `RELEASE_DB.CORE.V_ELIGIBLE_CHANGES` | View | Changes eligible for the next release |
| `RELEASE_DB.CORE.V_APPROVED_FOR_DEPLOY` | View | Handoff interface for the CI/CD Agent |
| `RELEASE_DB.CORE.RUN_RELEASE_CYCLE` | Task | Starts release preparation |
| `RELEASE_DB.CORE.PROCESS_DECISIONS` | Task | Applies gatekeeper decisions |
| `RELEASE_DB.CORE.RELEASE_AGENT` | Cortex Agent | Orchestration and conversational interface |
| `RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG` | Table | Append-only event history |
| `RELEASE_DB.AUDIT.RELEASE_LINEAGE_SNAPSHOT` | Table | Lineage evidence captured at submission |
| `RELEASE_DB.AUDIT.V_RELEASE_EVIDENCE` | Secure View | Auditor-facing release evidence |
| `RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC` | Cortex Search | Searchable release history |

---

## Roles and Access

| Role | Access |
|---|---|
| `RELEASE_AGENT_ROLE` | Read/write on core tables, insert-only on audit, Cortex usage |
| `RELEASE_GATEKEEPER_ROLE` | Insert decisions, read packages and notes |
| `RELEASE_AUDITOR_ROLE` | Read-only on secure audit views and the search service |

The agent role can never write `APPROVED` events. Only humans approve releases.

---

## Configuration

| Setting | Default | Location |
|---|---|---|
| Release cycle frequency | `15 MINUTE` | `RUN_RELEASE_CYCLE` schedule |
| Decision processing frequency | `5 MINUTE` | `PROCESS_DECISIONS` schedule |
| Notes model | `claude-sonnet-4-5` | `SP_DRAFT_RELEASE_NOTES` |
| Notes temperature | `0.1` | `SP_DRAFT_RELEASE_NOTES` |
| Search target lag | `1 hour` | `RELEASE_HISTORY_SVC` |
| Audit retention tag | `7Y` | `RETENTION_POLICY` tag |
| Re-check dependencies after | `24 hours` waiting | `SP_APPLY_DECISIONS` |
| Notification channel | `RELEASE_SLACK_INT` | `SP_SUBMIT_TO_GATEKEEPER` |

---

## Naming Conventions

- **Release ID:** `REL-<ENV>-<YYYYMMDD>-<SEQ>`, for example `REL-PROD-20261110-001`
- **Change ID:** `CHG-<TICKET>-<SHORT_SHA>`
- **Release version:** semantic versioning, `MAJOR.MINOR.PATCH`
- **Objects:** `UPPER_SNAKE_CASE`; procedures prefixed `SP_`, views `V_`, streams `_STREAM`, search services `_SVC`
- **JSON keys:** `snake_case`

---

## Monitoring

```sql
-- Task health
SELECT NAME, STATE, SCHEDULED_TIME, COMPLETED_TIME, ERROR_MESSAGE
FROM TABLE(INFORMATION_SCHEMA.TASK_HISTORY())
WHERE NAME IN ('RUN_RELEASE_CYCLE', 'PROCESS_DECISIONS')
ORDER BY SCHEDULED_TIME DESC
LIMIT 20;

-- Packages waiting on the gatekeeper
SELECT RELEASE_ID, TARGET_ENV, DATEDIFF('hour', UPDATED_AT, CURRENT_TIMESTAMP()) AS HOURS_WAITING
FROM RELEASE_DB.CORE.RELEASE_PACKAGES
WHERE STATUS = 'SUBMITTED'
ORDER BY HOURS_WAITING DESC;

-- Release throughput (last 30 days)
SELECT STATUS, COUNT(*) AS RELEASES
FROM RELEASE_DB.CORE.RELEASE_PACKAGES
WHERE CREATED_AT >= DATEADD('day', -30, CURRENT_TIMESTAMP())
GROUP BY STATUS;
```

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
|---|---|---|
| Task never runs | Task suspended or stream empty | `ALTER TASK ... RESUME`; confirm PASSED rows exist |
| Package stuck in `DRAFT` | Blocking dependency or notes validation failed | Check `DEPENDENCY_REPORT:blocking` and `NOTES_VALIDATION_FAILED` audit events |
| Recently created objects show as missing dependencies | `ACCOUNT_USAGE` latency | Add `INFORMATION_SCHEMA` checks and parsed DDL from the repository |
| Notes mention objects that are not in scope | Prompt not grounded strictly enough | Tighten the template; keep temperature at or below 0.2 |
| No gatekeeper notification | Integration missing or not granted | Check `RELEASE_SLACK_INT` and grant usage to the agent role |
| Stale search results | Target lag or suspended indexing | `DESCRIBE CORTEX SEARCH SERVICE`; resume indexing |

---

## Pipeline Integration

| Direction | Agent | Interface |
|---|---|---|
| Upstream | Testing Agent, Reconciliation Agent | Write to `RELEASE_CANDIDATES` |
| Downstream | CI/CD Agent | Reads `V_APPROVED_FOR_DEPLOY` |
| Downstream | Deployment Validation Agent | Sets `RELEASED` and logs the result |
| Consumers | Observability Agent, Remediation Agent | Query `RELEASE_HISTORY_SVC` for change context |

---

## Guardrails

- Only changes with `TEST_STATUS = 'PASSED'` enter a release.
- A package with unresolved blocking dependencies never reaches the gatekeeper.
- The agent prepares and recommends; humans approve.
- Every state change writes an audit event.
- No sensitive data values go into audit details, notes or the search index.
- All work runs under `RELEASE_AGENT_ROLE` on `RELEASE_WH`, never `ACCOUNTADMIN`.