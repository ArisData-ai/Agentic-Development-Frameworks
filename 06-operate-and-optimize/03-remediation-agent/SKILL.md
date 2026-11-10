---
name: remediation-agent
description: This skill defines the end-to-end Remediation Agent for Snowflake, which detects triaged operational incidents, diagnoses likely root cause, drafts fixes or runbook steps, stages safe remediations, and routes proposals to an Operations Lead through Cortex Agents, Cortex Search, Streams, Tasks, and Alerts. Use this skill for any task involving incident remediation design, automated recovery for failed tasks or stalled pipelines, breached SLA thresholds, on-call diagnosis workflows, MTTR reduction, human-in-the-loop approval gates, or building a remediation agent as part of an Operate and Optimize agent segment.
---
# Remediation Agent

## Overview
The Remediation Agent shortens time to recovery for always-on data products. It senses triaged incidents, reasons over telemetry and historical resolutions, proposes a fix, executes only pre-approved safe actions automatically, and hands everything else to the Operations Lead for approval, modification, or blocking. It also records outcomes so recurring causes surface for permanent fixes.

Core loop:

    SENSE    -> triaged incident (failed task, stalled pipeline, breached threshold)
    DIAGNOSE -> telemetry + Cortex Search (runbooks, past incidents) + Cortex Agent reasoning
    PROPOSE  -> root cause, confidence, fix or runbook step, risk tier
    ACT      -> auto-execute SAFE tier only (retry, resume, refresh)
    HAND OFF -> Operations Lead approves / modifies / blocks everything else
    LEARN    -> log outcome, cluster recurring causes, feed runbook corpus

## When to Use
- Designing or deploying the full remediation workflow in a Snowflake account
- Wiring the agent to upstream observability or triage outputs
- Defining the shared schema, roles, and objects every sibling skill depends on
- Explaining agent scope, benefits, and ideal customer profile to stakeholders

## Instructions
1. Provision a dedicated database, schema, warehouse, and role hierarchy before building any component.

        CREATE DATABASE IF NOT EXISTS OPS_DB;
        CREATE SCHEMA IF NOT EXISTS OPS_DB.REMEDIATION;
        CREATE WAREHOUSE IF NOT EXISTS REMEDIATION_WH
            WAREHOUSE_SIZE = 'XSMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;

        CREATE ROLE IF NOT EXISTS REMEDIATION_AGENT_ROLE;   -- reads telemetry, writes proposals
        CREATE ROLE IF NOT EXISTS REMEDIATION_EXECUTOR_ROLE; -- runs approved actions only
        CREATE ROLE IF NOT EXISTS OPERATIONS_LEAD_ROLE;      -- approves, modifies, blocks

        GRANT USAGE ON DATABASE OPS_DB TO ROLE REMEDIATION_AGENT_ROLE;
        GRANT USAGE ON SCHEMA OPS_DB.REMEDIATION TO ROLE REMEDIATION_AGENT_ROLE;
        GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE REMEDIATION_AGENT_ROLE;

2. Create the core incident and proposal tables shared by all sibling skills.

        CREATE TABLE IF NOT EXISTS OPS_DB.REMEDIATION.INCIDENTS (
            incident_id      STRING DEFAULT UUID_STRING(),
            source_type      STRING,          -- TASK | PIPE | DYNAMIC_TABLE | STREAM | THRESHOLD
            object_name      STRING,
            severity         STRING,          -- SEV1..SEV4
            error_code       STRING,
            error_message    STRING,
            detected_at      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
            status           STRING DEFAULT 'OPEN',
            context          VARIANT
        );

        CREATE TABLE IF NOT EXISTS OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS (
            proposal_id      STRING DEFAULT UUID_STRING(),
            incident_id      STRING,
            root_cause       STRING,
            confidence       FLOAT,
            action_type      STRING,          -- RETRY_TASK | RESUME_TASK | REFRESH_DT | CUSTOM_SQL | RUNBOOK_STEP
            action_sql       STRING,
            risk_tier        STRING,          -- SAFE | REVIEW | BLOCKED
            evidence         VARIANT,
            status           STRING DEFAULT 'PENDING',
            decided_by       STRING,
            decided_at       TIMESTAMP_LTZ,
            executed_at      TIMESTAMP_LTZ,
            outcome          STRING
        );

3. Build components in this order, each covered by a sibling skill: incident intake, runbook retrieval, diagnosis, safe action execution, approval handoff, recurrence tracking.
4. Orchestrate the loop with a scheduled Task or Alert that calls the diagnosis procedure for every OPEN incident.
5. Enforce the rule that the agent never executes REVIEW tier actions without a recorded Operations Lead decision.
6. Validate end to end in a non-production account by forcing a task failure and confirming the proposal, approval, execution, and outcome rows.

## Coding Conventions
- Use fully qualified object names (DATABASE.SCHEMA.OBJECT) in all SQL.
- Use UPPER_SNAKE_CASE for objects and lower_snake_case for columns.
- Keep agent reasoning, execution, and approval in separate roles; never grant OWNERSHIP of production objects to the agent role.
- Store every proposal and action in REMEDIATION_PROPOSALS; no side-channel execution.
- Prefer idempotent DDL (CREATE IF NOT EXISTS, CREATE OR REPLACE for procedures).

## References
- references/remediation-incident-intake.md: detection of failed tasks, stalled pipelines, stale streams, and breached thresholds into the INCIDENTS table
- references/remediation-runbook-retrieval.md: Cortex Search service over runbooks and past incident resolutions
- references/remediation-root-cause-diagnosis.md: Cortex Agent configuration that produces root cause, confidence, and proposed fix
- references/remediation-safe-action-execution.md: pre-approved action allowlist and the executor procedure for retries, resumes, and refreshes
- references/remediation-approval-handoff.md: Operations Lead notification, approve, modify, and block workflow
- references/remediation-recurrence-tracking.md: outcome logging, recurring cause analysis, and MTTR reporting