# Observability Agent

*Segment 6: Operate & Optimize*

The Observability Agent watches the health and performance of Snowflake pipelines and workloads so problems show up early. It detects latency, failure, freshness and volume anomalies, groups related alerts into a single incident, ranks incidents by business impact, and sends a triaged view to the Operations Lead, who confirms the priority and assigns the response.

## Why It Exists

Organizations with many pipelines often get far more alerts than anyone can act on. The agent cuts that noise down to a short, ranked list of incidents, which shortens the time it takes to detect real problems.

| Audience | Benefit |
|---|---|
| Operations teams | Work from prioritized incidents instead of raw alerts |
| Data product owners | Hear about problems before their consumers do |
| Leadership | See platform reliability trends over time |

## How It Works

    Sense  -->  Act  -->  Hand off
    
    Sense     Detect latency, failure, freshness and volume anomalies
              from Task history, Dynamic Table refresh history,
              Snowpipe copy history and query history.
    
    Act       Deduplicate signals, cluster them by time window and
              upstream lineage, select the root cause, score business
              impact and generate a Cortex triage summary.
    
    Hand off  Escalate prioritized incidents to the Operations Lead,
              record acknowledgment, confirmed priority and assignee,
              then expose acknowledged incidents to the Remediation Agent.

## Snowflake Components

| Component | Role in the agent |
|---|---|
| Streams & Tasks | Pipeline execution events; orchestration of the agent loop |
| Dynamic Tables | Refresh health, target lag and freshness signals |
| ACCOUNT_USAGE / INFORMATION_SCHEMA | Task, refresh, copy and query telemetry |
| SNOWFLAKE.ML.ANOMALY_DETECTION | Learned baselines for seasonal volume |
| Cortex COMPLETE / Cortex Agents | Signal correlation, triage summaries, conversational triage |
| Snowflake Alerts & Notification Integrations | Email, Slack, Teams and webhook escalation |

## Folder Structure

    observability-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── observability-signal-detection.md
        └── observability-incident-correlation.md
        └── observability-escalation-handoff.md
        └── observability-reliability-reporting.md

## Core Objects

All agent objects live in `OBS_DB.MONITOR`.

| Object | Purpose |
|---|---|
| `PIPELINE_REGISTRY` | Monitored pipelines with owner, business tier and SLO thresholds |
| `SIGNAL_EVENTS` | Normalized anomaly signals with evidence |
| `INCIDENTS` | Correlated, prioritized incidents with triage summaries |
| `ESCALATIONS` | Notification log, acknowledgment, confirmed priority and assignee |
| `SUPPRESSION_WINDOWS` | Maintenance and deployment windows that mute notifications |
| `SP_DETECT_SIGNALS` / `SP_CORRELATE_INCIDENTS` / `SP_ESCALATE_INCIDENTS` | Stage procedures |
| `SP_ACK_INCIDENT` | Operations Lead acknowledgment and assignment |
| `T_OBS_ROOT` Task graph | Runs detect, correlate and escalate every 5 minutes |
| `V_*` reporting views | MTTD, MTTA, noise reduction, data product health, reliability trend |

## Prerequisites

- A role with `IMPORTED PRIVILEGES` on the `SNOWFLAKE` database and `MONITOR EXECUTION` on the account
- A dedicated XSMALL warehouse (`OBS_WH`) with `AUTO_SUSPEND = 60`
- Cortex functions enabled in the account region, or cross-region inference turned on
- Email and/or webhook notification integrations approved by an account administrator
- At least 4 weeks of load history before you turn on ML volume baselines

## Quick Start

1. Create `OBS_DB.MONITOR`, `OBS_WH`, `OBS_AGENT_ROLE` and `OPS_LEAD_ROLE` (see `observability-agent`).
2. Add pipelines to `PIPELINE_REGISTRY` with tier, owner and SLO thresholds.
3. Deploy the detection procedures (`observability-signal-detection`).
4. Deploy correlation and impact scoring (`observability-incident-correlation`).
5. Configure notification integrations, alerts and the acknowledgment procedure (`observability-escalation-handoff`).
6. Resume the Task graph, starting with the child Tasks and ending with the root Task.
7. Validate in a non-production account: force one failed Task, one stale Dynamic Table and one empty load, then confirm that exactly one incident per root cause reaches the Operations Lead.
8. Publish the reporting views and build dashboards (`observability-reliability-reporting`).

## Using with Cortex Code

Start with `SKILL.md` for end-to-end setup, then load the sibling skill for the stage you are working on. Example prompts:

- "Set up the observability schema and pipeline registry for our Dynamic Tables."
- "Add freshness and volume detection for the finance data product pipelines."
- "Group duplicate alerts from one upstream failure into a single incident."
- "Send P1 incidents to our Slack channel and track Operations Lead acknowledgment."
- "Build a monthly reliability dashboard showing MTTD and noise reduction."

## Guardrails

- The agent observes, correlates and escalates; it never restarts, suspends or modifies the pipelines it monitors.
- Only `OPS_LEAD_ROLE` can confirm priority and assign incidents.
- Corrective action belongs to the Remediation Agent, through `V_REMEDIATION_QUEUE`.
- Webhook URLs and tokens live only in `SECRET` objects.
- Thresholds, grouping windows and impact weights live in config tables, not in code.

## Key Metrics

| Metric | Definition |
|---|---|
| MTTD | Minutes from first signal to incident creation |
| MTTA | Minutes from escalation to Operations Lead acknowledgment |
| Noise reduction ratio | 1 minus incidents divided by raw signals |
| SLO attainment | Share of time within target lag or freshness threshold |
| Priority override rate | Share of incidents where the confirmed priority differs from the agent's priority |

## Ideal Customer Profile

Organizations with many pipelines and alert-heavy operations, where teams struggle to separate critical incidents from noise.