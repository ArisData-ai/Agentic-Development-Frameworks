# Remediation Agent

*Segment 6: Operate & Optimize*

The Remediation Agent diagnoses operational incidents in Snowflake and prepares fixes so recovery can start right away. It picks up triaged incidents, finds the likely root cause, drafts a fix or runbook step, and runs only pre-approved safe actions such as retries and restarts. Anything riskier goes to the Operations Lead, who approves, modifies, or blocks it.

## Why It Matters

- **Faster recovery:** On-call engineers start with a diagnosis and a proposed fix instead of a blank screen.
- **Fewer repeat incidents:** Recurring causes are tracked and flagged for permanent fixes.
- **Controlled automation:** Only allowlisted, low-risk actions run without approval.

## Who Benefits

| Persona | Value |
|---|---|
| On-call engineers | Get a diagnosis and proposed fix the moment an incident opens |
| Service owners | Get data products back online faster |
| Operations managers | Track MTTR, auto-resolution rate, and recurring causes |

## How It Works

    SENSE    -> Failed tasks, stalled pipelines, stale streams, breached thresholds
    DIAGNOSE -> Telemetry + Cortex Search (runbooks, past incidents) + Cortex Agent reasoning
    PROPOSE  -> Root cause, confidence, fix or runbook step, risk tier
    ACT      -> Auto-execute SAFE tier only (retry, resume, refresh)
    HAND OFF -> Operations Lead approves, modifies, or blocks REVIEW tier actions
    LEARN    -> Log outcomes, cluster recurring causes, enrich runbook corpus

## Snowflake Components

| Component | Role in the Agent |
|---|---|
| Alerts | Detect incidents on a schedule and write them to `INCIDENTS` |
| Streams & Tasks | Trigger and run approved retries and recovery steps |
| Cortex Search | Retrieves runbooks and past incident resolutions |
| Cortex Agents | Plan diagnosis and produce structured remediation proposals |
| Stored Procedures | Run actions and record approval decisions with an audit trail |
| Notification Integrations | Notify the Operations Lead by email or webhook |
| Streamlit in Snowflake | Approval console for pending proposals |

## Folder Structure

    remediation-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── remediation-incident-intake.md
        └── remediation-runbook-retrieval.md
        └── remediation-root-cause-diagnosis.md
        └── remediation-safe-action-execution.md
        └── remediation-approval-handoff.md
        └── remediation-recurrence-tracking.md

## Prerequisites

- Snowflake account with Cortex Search and Cortex Agents available in your region
- A role able to create databases, warehouses, roles, alerts, and tasks
- `IMPORTED PRIVILEGES` on the `SNOWFLAKE` database for `ACCOUNT_USAGE` access
- Runbooks and past incident resolutions available for ingestion (Confluence, ServiceNow, Jira, Git, or Markdown)
- A notification target for the Operations Lead (email, Slack, Microsoft Teams, or PagerDuty)

## Quick Start

1. **Provision the foundation:** create `OPS_DB.REMEDIATION`, `REMEDIATION_WH`, and the three roles from `remediation-agent`.
2. **Create core tables:** `INCIDENTS` and `REMEDIATION_PROPOSALS`.
3. **Enable intake:** deploy the failed-task alert and any pipeline, stream, or threshold checks from `remediation-incident-intake`.
4. **Load knowledge:** populate `KNOWLEDGE_CORPUS` and create `RUNBOOK_SEARCH` from `remediation-runbook-retrieval`.
5. **Deploy diagnosis:** create `REMEDIATION_AGENT` and the risk tiering rules from `remediation-root-cause-diagnosis`.
6. **Enable safe execution:** seed `SAFE_ACTION_ALLOWLIST`, then deploy `EXECUTE_PROPOSAL` and `TASK_EXECUTE_READY`.
7. **Wire approvals:** configure notifications, `DECIDE_PROPOSAL`, and the Streamlit console.
8. **Track outcomes:** deploy `V_REMEDIATION_METRICS` and the weekly recurrence task.
9. **Validate:** force a task failure in a non-production account and confirm the incident, proposal, decision, execution, and outcome rows.

## Risk Tiers

| Tier | Criteria | Execution |
|---|---|---|
| SAFE | Allowlisted action type and confidence of 0.80 or higher | Runs automatically within retry limits |
| REVIEW | Custom SQL, confidence below 0.80, or retry limit reached | Needs an Operations Lead decision |
| BLOCKED | Contains DROP, TRUNCATE, DELETE, GRANT, or REVOKE | Never runs |

## Guardrails

- The agent role never approves its own proposals and never owns production objects.
- SAFE actions run only from allowlist templates, never from raw model output.
- Per-object retry limits and cooldowns prevent retry storms.
- Every proposal, decision, and execution is logged in `REMEDIATION_PROPOSALS`.
- Runbook content must not contain secrets or credentials.

## Key Metrics

- **MTTR:** minutes from detection to successful execution, by severity
- **Auto-resolution rate:** share of incidents resolved by SAFE actions
- **Recurrence:** objects with three or more incidents of the same error in 30 days
- **Approval latency:** time proposals spend in PENDING before a decision

## Ideal Customer Profile

Teams that run always-on data products with tight SLAs, where manual incident response causes long outages and on-call fatigue.