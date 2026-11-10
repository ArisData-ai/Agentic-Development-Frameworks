---
name: reconciliation-agent
description: This skill defines the end-to-end Reconciliation Agent workflow that detects completed loads and migration cutover milestones, compares source and target data on row counts, control totals, and key fields, pinpoints variances, and routes reconciliation reports to the QA Lead for sign-off, built on Snowpark, Streams and Tasks, Snowflake Horizon, and Cortex Agents. Use this skill for any task involving migration reconciliation, source-to-target validation, data tie-out, cutover readiness evidence, audit-ready reconciliation reports, or designing a reconciliation control framework when moving financial or operational data from Oracle, SQL Server, Teradata, SAP, or other legacy ERP systems into Snowflake.
---

# Reconciliation Agent

## Overview
The Reconciliation Agent confirms that migrated or transformed data matches its source. It belongs to Segment 4 (Test & Validate) of the AI-agent delivery pipeline and produces documented, source-to-target evidence that lets stakeholders trust the new platform and approve cutover.

The agent runs a four-stage loop:

    SENSE   ->  Detect completed loads and migration cutover milestones (Streams & Tasks)
    COMPARE ->  Compare row counts, control totals, and key fields (Snowpark)
    ANALYZE ->  Pinpoint and classify mismatches; trace them through lineage (Snowflake Horizon)
    HAND OFF -> Send reconciliation reports to the QA Lead for sign-off or investigation (Cortex Agents)

The agent never signs off its own results. The QA Lead is the single human decision point for every variance and every cutover gate.

Primary beneficiaries:
- Finance and controllers: proof that figures tie out.
- Migration teams: early variance resolution.
- Auditors: documented source-to-target evidence.

Ideal customer profile: organizations migrating financial or operational data where unreconciled differences block cutover approval.

## When to Use
- Building a reconciliation framework for a legacy-to-Snowflake migration.
- Validating that a full, incremental, or CDC load landed completely and accurately.
- Producing tie-out evidence for finance close, regulatory reporting, or audit.
- Gating a cutover milestone (mock cutover, parallel run, go-live) on reconciliation status.
- Investigating why Snowflake totals differ from the legacy system.

## Instructions

1. Provision the control schema. Create a dedicated database and schemas for reconciliation metadata, separate from business data:

        CREATE DATABASE IF NOT EXISTS RECON_DB;
        CREATE SCHEMA IF NOT EXISTS RECON_DB.CONTROL;     -- config, audit, runs, results, variances
        CREATE SCHEMA IF NOT EXISTS RECON_DB.REPORTING;   -- views consumed by QA Lead and auditors
        CREATE WAREHOUSE IF NOT EXISTS RECON_WH
            WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;

2. Register every reconciled object in RECON_CONFIG with its keys, measures, compare columns, tolerances, and criticality. No object is reconciled without a config row.

3. Capture source evidence at extraction time. For each load, record source row counts and control totals (sums of amount and quantity columns) in LOAD_AUDIT. When field-level comparison is required, land a source snapshot in a RAW schema so both sides are queryable in Snowflake.

4. Wire the trigger. Use a stream on LOAD_AUDIT and a task chain to enqueue and execute reconciliation runs automatically (see reconciliation-trigger-detection).

5. Run tiered comparisons: row counts, then aggregate control totals, then hash-based key-field comparison. Short-circuit to PASS when table-level hashes match (see reconciliation-comparison-engine).

6. Classify and trace variances by type and likely root cause, apply tolerances and materiality, and attach upstream lineage (see reconciliation-variance-analysis).

7. Publish and hand off. Generate the reconciliation report, notify the QA Lead, and record the decision; block cutover until every in-scope object is PASS or SIGNED_OFF (see reconciliation-reporting-signoff).

Human-in-the-loop rules:
- The agent may queue, run, re-run, and classify. It may not alter source or target data, change tolerances, or record a sign-off.
- Tolerance and config changes require QA Lead approval and are versioned.
- Every decision is written to RECON_SIGNOFF with reviewer, timestamp, and rationale.

## Coding Conventions
- Object naming: RECON_DB.<SCHEMA>.<OBJECT>; tables in UPPER_SNAKE_CASE; procedures prefixed SP_, tasks TSK_, streams _STRM, views V_.
- Every run carries a RUN_ID (UUID_STRING()) propagated to results, variances, and sign-offs.
- Status vocabulary is fixed: QUEUED, RUNNING, PASS, WARN, FAIL, ERROR for runs; SIGNED_OFF, INVESTIGATE, REJECTED for decisions.
- All control tables are append-only; corrections are new rows, never updates to history.
- Use a dedicated role RECON_AGENT_ROLE with SELECT on source and target objects and write access only to RECON_DB.
- Parameterize object names through RECON_CONFIG; never hardcode table names inside procedures.
- Run reconciliation on a separate warehouse so costs and queueing are isolated from production loads.

## References
| Skill file | Covers |
|---|---|
| references/reconciliation-trigger-detection.md | Control tables, LOAD_AUDIT capture, streams, task chains, and milestone-driven run queuing |
| references/reconciliation-comparison-engine.md | Snowpark procedures for row-count, control-total, and hash-based key-field comparisons |
| references/reconciliation-variance-analysis.md | Variance classification, column-level drilldown, tolerance and materiality rules, lineage tracing with Snowflake Horizon |
| references/reconciliation-reporting-signoff.md | Report structure, QA Lead notification, sign-off capture, cutover readiness gates, and Cortex Agents escalation |