# Reconciliation Agent

*Segment 4: Test & Validate*

Proves that migrated or transformed data matches its source, and gives stakeholders the evidence they need to trust the new platform and approve cutover.

## Overview
The Reconciliation Agent watches for completed loads and migration cutover milestones. It compares source and target data on row counts, control totals, and key fields, finds the exact records and columns that don't match, and sends a reconciliation report to the **QA Lead**. The QA Lead signs off on each variance or sends it back for investigation.

The agent prepares the evidence. It never approves its own results.

    SENSE    ->  Detect completed loads and cutover milestones        (Streams & Tasks)
    COMPARE  ->  Row counts, control totals, key-field hashes          (Snowpark)
    ANALYZE  ->  Classify variances, trace lineage, rank by impact     (Snowflake Horizon)
    HAND OFF ->  Report to QA Lead for sign-off or investigation       (Cortex Agents)

## Primary Use Case
Proving migration accuracy before cutover, so unreconciled differences are found and resolved early, not after go-live.

## Who Benefits
| Audience | Value |
|---|---|
| Finance and controllers | Proof that figures tie out between legacy and Snowflake |
| Migration teams | Early, pinpointed variances with likely root causes |
| QA Leads | Ranked, evidence-backed findings for every decision |
| Auditors | Documented, reproducible source-to-target evidence |

## Ideal Customer Profile
Organizations migrating financial or operational data from Oracle, SQL Server, Teradata, SAP, or other legacy ERP systems, where unreconciled differences block cutover approval.

## Snowflake Platform Components
| Component | Role in the agent |
|---|---|
| Snowpark | Runs tiered comparison logic at scale inside Snowflake |
| Streams & Tasks | Detect completed loads and trigger reconciliation runs |
| Snowflake Horizon | Preserves lineage and access history for variance tracing |
| Cortex Agents | Coordinate comparisons, Q&A, re-runs, and escalations |
| Cortex Analyst | Lets the QA Lead query reconciliation results in natural language |
| Alerts and Notification Integrations | Notify the QA Lead when high-criticality objects fail |

## Skill Files
| Skill | Covers |
|---|---|
| `SKILL` | Primary skill: end-to-end workflow, human-in-the-loop rules, shared conventions |
| `reconciliation-trigger-detection.md` | Control tables, load audit capture, streams, task chains, milestone queuing |
| `reconciliation-comparison-engine.md` | Snowpark procedure for row-count, control-total, and key-field comparison |
| `reconciliation-variance-analysis.md` | Variance classification, column drilldown, tolerances, lineage tracing |
| `reconciliation-reporting-signoff.md` | Reports, alerts, QA Lead sign-off, cutover readiness gates |

## Folder Structure

    reconciliation-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── reconciliation-trigger-detection.md
        └── reconciliation-comparison-engine.md
        └── reconciliation-variance-analysis.md
        └── reconciliation-reporting-signoff.md

## Data Model
All reconciliation metadata lives in `RECON_DB`, separate from business data.

| Object | Purpose |
|---|---|
| `CONTROL.RECON_CONFIG` | One row per source-target pair: keys, measures, compare columns, tolerances, criticality |
| `CONTROL.LOAD_AUDIT` | Completed loads with source row counts and control totals captured at extraction |
| `CONTROL.RECON_RUN` | Run queue and status per load or milestone |
| `CONTROL.RECON_RESULT` | Per-check outcomes: row count, control totals, key-field |
| `CONTROL.RECON_VARIANCE` | Record-level mismatches with variance type and root cause hint |
| `CONTROL.RECON_SIGNOFF` | QA Lead decisions with reviewer, timestamp, and rationale |
| `REPORTING.V_RECON_SUMMARY` | Per-run, per-object status for review |
| `REPORTING.V_CUTOVER_READINESS` | READY or BLOCKED gate per milestone |

## Quick Start
1. Create the control database, schemas, and warehouse:

        CREATE DATABASE IF NOT EXISTS RECON_DB;
        CREATE SCHEMA IF NOT EXISTS RECON_DB.CONTROL;
        CREATE SCHEMA IF NOT EXISTS RECON_DB.REPORTING;
        CREATE WAREHOUSE IF NOT EXISTS RECON_WH
            WAREHOUSE_SIZE = 'SMALL' AUTO_SUSPEND = 60 AUTO_RESUME = TRUE;

2. Create roles and grant least-privilege access:

        CREATE ROLE IF NOT EXISTS RECON_AGENT_ROLE;   -- reads source/target, writes RECON_DB.CONTROL
        CREATE ROLE IF NOT EXISTS QA_LEAD_ROLE;       -- reads reports, writes RECON_SIGNOFF

3. Deploy the control tables, stream, and task chain from `reconciliation-trigger-detection`.
4. Deploy `SP_RUN_QUEUED_RECONS` from `reconciliation-comparison-engine`.
5. Register in-scope objects in `RECON_CONFIG` and have the QA Lead approve the tolerances.
6. Update loaders (Openflow, Snowpipe, COPY INTO, dbt, or custom ELT) to write a `LOAD_AUDIT` row, with source-side counts and totals, when each load completes.
7. Deploy the reporting views, alert, and Cortex Agent from `reconciliation-reporting-signoff`.
8. Resume the tasks and alert:

        ALTER TASK RECON_DB.CONTROL.TSK_RECON_EXECUTE RESUME;
        ALTER TASK RECON_DB.CONTROL.TSK_RECON_TRIGGER RESUME;
        ALTER ALERT RECON_DB.CONTROL.ALR_RECON_FAIL RESUME;

## Run a Reconciliation Manually
Queue a re-run for one object:

    INSERT INTO RECON_DB.CONTROL.RECON_RUN
        (RUN_ID, LOAD_ID, CONFIG_ID, MILESTONE, TRIGGER_TYPE, STATUS, CREATED_AT)
    VALUES (UUID_STRING(), '<LOAD_ID>', '<CONFIG_ID>', NULL, 'MANUAL', 'QUEUED', CURRENT_TIMESTAMP());

    CALL RECON_DB.CONTROL.SP_RUN_QUEUED_RECONS();

Check results and cutover readiness:

    SELECT * FROM RECON_DB.REPORTING.V_RECON_SUMMARY ORDER BY ENDED_AT DESC;
    SELECT * FROM RECON_DB.REPORTING.V_CUTOVER_READINESS;

## Human-in-the-Loop Controls
| Action | Agent | QA Lead |
|---|---|---|
| Queue, run, and re-run reconciliations | Yes | Yes |
| Classify variances and assign root cause hints | Yes | Reviews |
| Change tolerances or config | No | Approves |
| Record SIGNED_OFF, INVESTIGATE, or REJECTED | No | Yes |
| Modify source or target data | No | No (engineering only) |

Privileges enforce these controls. `RECON_AGENT_ROLE` has SELECT only on `RECON_SIGNOFF`, so the agent can't sign off on anything.

## Status Vocabulary
| Type | Values |
|---|---|
| Run and check status | QUEUED, RUNNING, PASS, WARN, FAIL, ERROR |
| QA Lead decision | SIGNED_OFF, INVESTIGATE, REJECTED |
| Variance type | MISSING_IN_TARGET, EXTRA_IN_TARGET, VALUE_MISMATCH |
| Cutover gate | READY, BLOCKED |

## Conventions
- Compute source control totals in the source system at extraction. Never derive them from the Snowflake target.
- Before comparing, normalize data types, trimming, timezones, and decimal scale on both sides.
- Control tables are append-only. Corrections go in as new rows, and config changes are versioned.
- Deterministic rules set every status. AI-generated summaries explain results but never change them.
- Variance tables store keys and column names only, never sensitive field values.
- Reconciliation runs on a dedicated warehouse, which keeps its cost and queueing separate from production loads.

## Pipeline Context
| Upstream | This agent | Downstream |
|---|---|---|
| Pipeline Agent, Code Generation Agent (Segment 3) | Reconciliation Agent (Segment 4) | Release Agent, CI/CD Agent (Segment 5) |

Sibling agents in Segment 4: **Data Quality Agent** and **Testing Agent**.