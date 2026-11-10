# Data Quality Agent

*Segment 4: Test & Validate | AI Agents for the Whole Project Pipeline*

The Data Quality Agent checks Snowflake data against agreed quality rules, so problems are caught before anyone downstream uses the data. It spots new loads, runs freshness, completeness, uniqueness, validity, and distribution checks, suggests new rules where coverage is missing, and quarantines records that fail. It then sends a summary with the evidence to the **QA Lead**, who approves rule changes and decides whether to release or hold the affected data.

---

## Why This Agent

| Problem | What the agent does |
|---|---|
| Bad data reaches reports before anyone notices | Checks every load and isolates failing records first |
| Quality rules are incomplete or out of date | Profiles the data and suggests rules based on what it actually contains |
| Data quality decisions leave no record | Logs every check, hold, release, and approval with who did it and when |
| Too many low-priority alerts | Groups results by severity and escalates only what needs a decision |

**Ideal customer profile:** Organizations where bad data has caused reporting errors or lost trust, such as finance, retail, and ERP-driven analytics teams.

---

## Who Benefits

- **Data stewards:** See issues as soon as they appear, not after users complain.
- **Analysts:** Use data that has passed documented checks.
- **QA leads:** Make quality decisions with the evidence in front of them.

---

## How It Works

    SENSE     Streams and triggered Tasks detect new loads, freshness breaches, and volume anomalies
       |
    CHECK     Horizon data metric functions evaluate ACTIVE rules; results written per rule and run
       |
    PROPOSE   Snowpark profiling finds coverage gaps; new rules inserted as PROPOSED only
       |
    QUARANTINE Failing BLOCKER and MAJOR records isolated; only passing rows publish
       |
    ESCALATE  Evidence package sent to the QA Lead, who approves rules and releases or holds data

A Cortex Agent coordinates these steps. Each step is a stored procedure registered as a tool.

---

## Snowflake Components

| Component | Role |
|---|---|
| Snowflake Horizon | System and custom data metric functions, classification, masking, monitoring results |
| Streams & Tasks | Event-driven load detection and scheduled sweeps |
| Snowpark | Column profiling and distribution drift analysis |
| Cortex Agents | Plans the checks and routes exceptions |
| Cortex Analyst | Lets stakeholders ask plain-language questions about pass rates and decisions (optional) |
| Notification Integrations | Email or webhook alerts to the QA Lead |

---

## Skill Files

| Skill | Covers |
|---|---|
| `SKILL` | Shared schema, roles, workflow order, Cortex Agent specification |
| `dq-load-sensing` | Streams, triggered tasks, freshness and volume detection |
| `dq-check-execution` | System and custom DMFs, scheduling, threshold evaluation |
| `dq-rule-proposal` | Profiling, coverage gaps, proposed rule generation |
| `dq-quarantine-management` | Write-audit-publish, quarantine, release mechanics |
| `dq-qa-lead-escalation` | Evidence packages, notifications, decision audit trail |

---

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── dq-load-sensing.md
        └── dq-check-execution.md
        └── dq-rule-proposal.md
        └── dq-quarantine-management.md
        └── dq-qa-lead-escalation.md

---

## Prerequisites

- A Snowflake edition that supports data metric functions (Enterprise or higher)
- Cortex Agents available in your account region
- ACCOUNTADMIN or an equivalent role for the initial setup (roles, grants, notification integration)
- A warehouse for agent workloads (an XS warehouse is recommended, for example `DQ_SVC_WH`)
- QA Lead email addresses, or a webhook for Slack or Teams alerts

---

## Quick Start

**1. Create the foundation**

    snow sql -f sql/00_setup_database_roles.sql
    snow sql -f sql/01_config_tables.sql
    snow sql -f sql/02_results_tables.sql
    snow sql -f sql/03_quarantine_tables.sql

**2. Register a table to monitor**

    INSERT INTO DQ_DB.CONFIG.MONITORED_TABLES
        (TABLE_FQN, DATA_OWNER, CRITICALITY, FRESHNESS_SLA_MIN, PRIMARY_KEY_COLS, CONSUMER_SCHEMA)
    SELECT 'RAW_DB.ERP.SALES_ORDERS', 'finance.data@company.com', 'CRITICAL', 60,
           ARRAY_CONSTRUCT('ORDER_ID'), 'ANALYTICS_DB.SALES';

**3. Deploy the checks, procedures, and triggers**

    snow sql -f sql/04_custom_dmfs.sql
    snow sql -f sql/05_procedures.sql
    snow sql -f sql/06_streams_tasks.sql
    snow sql -f sql/07_notifications.sql
    snow sql -f sql/08_agent.sql

**4. Generate starter rules**

    CALL DQ_DB.OPS.SP_PROPOSE_RULES('RAW_DB.ERP.SALES_ORDERS');

**5. QA Lead approves the rules**

    USE ROLE DQ_QA_LEAD;
    SELECT * FROM DQ_DB.CONFIG.DQ_RULES WHERE STATUS = 'PROPOSED';
    CALL DQ_DB.OPS.SP_APPROVE_RULE('<rule_id>');

**6. Load data and watch the agent run**

    SELECT * FROM DQ_DB.RESULTS.DQ_RUN_LOG ORDER BY CREATED_AT DESC;
    SELECT * FROM DQ_DB.RESULTS.DQ_CHECK_RESULTS WHERE RUN_ID = '<run_id>';
    SELECT * FROM DQ_DB.RESULTS.DQ_ESCALATIONS WHERE STATUS = 'PENDING';

---

## Severity and Actions

| Severity | Data action | Escalation |
|---|---|---|
| BLOCKER | Quarantine failing rows, or hold the entire load | P1, immediate notification |
| MAJOR | Quarantine failing rows | P2, decision within 1 business day |
| MINOR | Publish and log | P3, included in the daily digest |

---

## Key Tables

| Table | Purpose |
|---|---|
| `DQ_DB.CONFIG.MONITORED_TABLES` | Tables in scope, with owner, SLA, and criticality |
| `DQ_DB.CONFIG.DQ_RULES` | System of record for rules and their PROPOSED or ACTIVE status |
| `DQ_DB.CONFIG.COLUMN_BASELINES` | Profiling snapshots that back rule thresholds |
| `DQ_DB.RESULTS.DQ_RUN_LOG` | One row per detected load or trigger |
| `DQ_DB.RESULTS.DQ_CHECK_RESULTS` | Pass or fail result per rule per run |
| `DQ_DB.QUARANTINE.QUARANTINED_RECORDS` | Failing rows, with the rule and reason |
| `DQ_DB.QUARANTINE.HELD_LOADS` | Loads held in full |
| `DQ_DB.RESULTS.DQ_ESCALATIONS` | Evidence packages and QA Lead decisions |

---

## Roles and Guardrails

| Role | Can do | Cannot do |
|---|---|---|
| `DQ_AGENT_ROLE` | Read monitored tables, run checks, write results, quarantine records, propose rules | Activate rules, release held data, delete source records |
| `DQ_ENGINEER` | Build and maintain checks, procedures, and tasks | Make QA decisions |
| `DQ_QA_LEAD` | Approve, reject, or change rules; release, hold, or discard data | (Human approver; owns all decisions) |

- Held data is never released automatically, even when an escalation expires.
- Quarantine payloads inherit masking and row access policies from the source.
- Notifications include record keys and counts only, never sensitive values.

---

## Configuration

| Setting | Location | Default |
|---|---|---|
| Freshness SLA | `MONITORED_TABLES.FRESHNESS_SLA_MIN` | Set per table |
| Volume anomaly band | `DQ_DB.CONFIG` | Mean ± 3 standard deviations over the last 14 loads |
| Profiling window | `SP_PROPOSE_RULES` parameter | 30 days or 1M rows |
| Freshness sweep schedule | `TSK_FRESHNESS_SWEEP` | Every 30 minutes |
| Discarded record retention | Purge task | 90 days |

---


## Pipeline Position

| Upstream | This agent | Downstream |
|---|---|---|
| Pipeline Agent and Code Generation Agent (Segment 3) deliver loaded data | **Data Quality Agent** | Testing Agent and Reconciliation Agent (Segment 4), then the Release Agent (Segment 5) |

---

## Troubleshooting

| Symptom | Check |
|---|---|
| No runs created | Is the stream consumed inside the task? Is the task resumed? Does the role have USAGE on the warehouse? |
| DMF results missing | Is `DATA_METRIC_SCHEDULE` set on the table? Does the role hold `DATA_METRIC_USER`? |
| Quarantine count does not match the failed rows | The load is held by design; check the scope filter in the rule |
| QA Lead cannot release data | Run `IS_ROLE_IN_SESSION('DQ_QA_LEAD')` in the QA Lead's session |
| No email received | Is the recipient listed in `ALLOWED_RECIPIENTS` on `DQ_EMAIL_INT`? |