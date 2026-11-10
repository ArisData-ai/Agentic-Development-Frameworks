---
name: data-quality-agent
description: This skill defines the end-to-end Data Quality Agent that senses new loads, runs freshness, completeness, uniqueness, and distribution checks, proposes rules for coverage gaps, quarantines failing records, and escalates results to the QA Lead, built on Snowflake Horizon data metric functions, Streams and Tasks, and Cortex Agents. Use this skill for any task involving designing, deploying, or extending a data quality agent, such as setting up quality monitoring on Snowflake tables, orchestrating DQ workflows, defining quality thresholds, wiring QA Lead approval gates, onboarding ERP or CRM sources to quality checks, or building data trust controls for finance, retail, and analytics pipelines.
---
# Data Quality Agent

## Overview
The Data Quality Agent (Segment 4: Test & Validate) watches data against agreed quality rules so defects are caught before they reach consumers. It runs a closed loop:

    SENSE   -> detect new loads and shifts in freshness, completeness, uniqueness, distribution
    ACT     -> run checks, propose rules for gaps, quarantine failing records
    HAND OFF-> escalate results to the QA Lead, who approves rule changes and releases or holds data

The agent never publishes held data, never activates its own proposed rules, and never deletes source records. All final decisions belong to the QA Lead.

## When to Use
- Standing up quality monitoring for a new or migrated Snowflake data product
- Orchestrating the full sense, check, quarantine, escalate workflow
- Defining the shared schema, roles, and agent specification used by sibling skills
- Onboarding a new source (ERP, CRM, POS, finance ledgers) to quality controls
- Auditing whether quality controls work and data meets agreed thresholds

## Instructions

### 1. Provision the shared foundation
    CREATE DATABASE IF NOT EXISTS DQ_DB;
    CREATE SCHEMA IF NOT EXISTS DQ_DB.CONFIG;      -- rules, thresholds, table registry
    CREATE SCHEMA IF NOT EXISTS DQ_DB.RESULTS;     -- run log, check results
    CREATE SCHEMA IF NOT EXISTS DQ_DB.QUARANTINE;  -- failing records
    CREATE SCHEMA IF NOT EXISTS DQ_DB.OPS;         -- procedures, tasks, streams, agent

    CREATE ROLE IF NOT EXISTS DQ_AGENT_ROLE;       -- agent service role, least privilege
    CREATE ROLE IF NOT EXISTS DQ_QA_LEAD;          -- human approver
    CREATE ROLE IF NOT EXISTS DQ_ENGINEER;         -- builds and maintains checks
    GRANT ROLE DQ_AGENT_ROLE TO ROLE DQ_ENGINEER;
    GRANT DATABASE ROLE SNOWFLAKE.DATA_METRIC_USER TO ROLE DQ_AGENT_ROLE;
    GRANT EXECUTE DATA METRIC FUNCTION ON ACCOUNT TO ROLE DQ_AGENT_ROLE;
    GRANT APPLICATION ROLE SNOWFLAKE.DATA_QUALITY_MONITORING_VIEWER TO ROLE DQ_QA_LEAD;

### 2. Register monitored tables and rules
    CREATE TABLE IF NOT EXISTS DQ_DB.CONFIG.MONITORED_TABLES (
        TABLE_FQN          STRING NOT NULL,
        DATA_OWNER         STRING,
        CRITICALITY        STRING,         -- CRITICAL | HIGH | STANDARD
        FRESHNESS_SLA_MIN  NUMBER,
        PRIMARY_KEY_COLS   ARRAY,
        CONSUMER_SCHEMA    STRING,         -- where passing data is published
        ACTIVE             BOOLEAN DEFAULT TRUE
    );

    CREATE TABLE IF NOT EXISTS DQ_DB.CONFIG.DQ_RULES (
        RULE_ID            STRING DEFAULT UUID_STRING(),
        TABLE_FQN          STRING NOT NULL,
        COLUMN_NAME        STRING,
        DIMENSION          STRING,         -- FRESHNESS | COMPLETENESS | UNIQUENESS | DISTRIBUTION | VALIDITY
        METRIC_FUNCTION    STRING,         -- system or custom DMF name
        OPERATOR           STRING,         -- <= | >= | = | BETWEEN
        THRESHOLD_LOW      FLOAT,
        THRESHOLD_HIGH     FLOAT,
        SEVERITY           STRING,         -- BLOCKER | MAJOR | MINOR
        STATUS             STRING DEFAULT 'PROPOSED',  -- PROPOSED | ACTIVE | REJECTED | RETIRED
        RATIONALE          STRING,
        PROPOSED_BY        STRING,
        APPROVED_BY        STRING,
        APPROVED_AT        TIMESTAMP_LTZ,
        CREATED_AT         TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 3. Run the workflow in order
1. Load sensing (dq-load-sensing): streams and triggered tasks detect new data and open a run in DQ_DB.RESULTS.DQ_RUN_LOG.
2. Check execution (dq-check-execution): attach and evaluate DMFs for every ACTIVE rule; write normalized results.
3. Rule proposal (dq-rule-proposal): profile columns without coverage; insert rules as PROPOSED only.
4. Quarantine (dq-quarantine-management): isolate failing records for BLOCKER and MAJOR failures; publish only passing rows.
5. Escalation (dq-qa-lead-escalation): package evidence, notify the QA Lead, apply their release, hold, or rule decisions.

### 4. Create the Cortex Agent orchestrator
Expose each phase as a stored procedure and register it as a tool. Check the specification keys against the current Cortex Agents documentation before deploying.

    CREATE OR REPLACE AGENT DQ_DB.OPS.DATA_QUALITY_AGENT
      COMMENT = 'Senses loads, runs DQ checks, proposes rules, quarantines, escalates to QA Lead'
      FROM SPECIFICATION
      $$
      models:
        orchestration: auto
      instructions:
        orchestration: >
          For each open run: execute quality checks, quarantine records failing BLOCKER or MAJOR
          rules, propose rules for uncovered columns as PROPOSED only, then escalate a summary to
          the QA Lead. Never activate rules, release held data, or delete records.
        response: Report run id, table, pass rate, failed rules, quarantined row count, and the decision required.
      tools:
        - tool_spec: { type: generic, name: run_quality_checks,   description: Evaluate active rules for a run }
        - tool_spec: { type: generic, name: quarantine_failures,  description: Isolate failing records for a run }
        - tool_spec: { type: generic, name: propose_rules,        description: Profile and propose rules for gaps }
        - tool_spec: { type: generic, name: escalate_to_qa_lead,  description: Create escalation and notify QA Lead }
      tool_resources:
        run_quality_checks:  { type: procedure, identifier: DQ_DB.OPS.SP_RUN_QUALITY_CHECKS }
        quarantine_failures: { type: procedure, identifier: DQ_DB.OPS.SP_QUARANTINE_FAILURES }
        propose_rules:       { type: procedure, identifier: DQ_DB.OPS.SP_PROPOSE_RULES }
        escalate_to_qa_lead: { type: procedure, identifier: DQ_DB.OPS.SP_ESCALATE_RUN }
      $$;

### 5. Enforce guardrails
- The agent role has SELECT on monitored tables, INSERT on RESULTS and QUARANTINE, and no DELETE on source schemas.
- Rule activation and data release procedures check CURRENT_ROLE() = 'DQ_QA_LEAD' or are owned by and granted only to DQ_QA_LEAD.
- Every action writes to DQ_RUN_LOG with RUN_ID for full traceability.

### 6. Output expectations
Each run produces: one DQ_RUN_LOG row, one DQ_CHECK_RESULTS row per rule evaluated, quarantined rows tagged with RUN_ID and RULE_ID, zero or more PROPOSED rules, and one escalation record when any BLOCKER or MAJOR rule fails or proposals exist.

## Coding Conventions
- Objects use UPPER_SNAKE_CASE; procedures prefix SP_, tasks prefix TSK_, streams prefix STR_, custom DMFs prefix DMF_.
- Fully qualify every object as DATABASE.SCHEMA.OBJECT.
- Use CREATE ... IF NOT EXISTS for tables and CREATE OR REPLACE only for stateless objects (procedures, DMFs, agents).
- Store timestamps as TIMESTAMP_LTZ; store IDs as UUID_STRING().
- Keep thresholds in DQ_RULES, never hard-coded in procedures.
- Parameterize warehouse names per environment (DEV, QA, PROD) through deployment variables.

## References
- references/dq-load-sensing.md: Streams, triggered tasks, freshness and volume detection, run log creation.
- references/dq-check-execution.md: Horizon system and custom DMFs, scheduling, threshold evaluation, results table.
- references/dq-rule-proposal.md: Column profiling, coverage gap detection, proposed rule generation.
- references/dq-quarantine-management.md: Quarantine tables, failing record isolation, publish of passing rows, release mechanics.
- references/dq-qa-lead-escalation.md: Escalation packaging, notifications, QA Lead approval workflow, decision audit trail.