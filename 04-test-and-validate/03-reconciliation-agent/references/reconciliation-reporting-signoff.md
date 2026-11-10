---
name: reconciliation-reporting-signoff
description: This skill defines how the Reconciliation Agent publishes reconciliation reports, notifies and escalates to the QA Lead, captures sign-off, investigate, or reject decisions with full audit trail, and enforces cutover readiness gates, coordinated by Cortex Agents with Snowflake Horizon retaining evidence for auditors. Use this skill for any task involving building reconciliation summary views, generating tie-out reports for finance or controllers, sending email or Slack alerts on failed reconciliations, recording QA Lead approvals, producing auditor evidence packs, or deciding whether a migration milestone is ready for go-live.
---

# Reconciliation Reporting and Sign-off

## Overview
This skill implements the HAND OFF stage. The agent prepares evidence and routes it; the QA Lead decides. Cutover is blocked until every in-scope object is PASS or explicitly SIGNED_OFF.

    QA Lead decisions:
        SIGNED_OFF   variance accepted with rationale; object cleared
        INVESTIGATE  variance requires engineering follow-up; re-run expected
        REJECTED     load invalid; reload required before re-reconciliation

## When to Use
- Publishing reconciliation outcomes after each run or milestone.
- Setting up notifications and escalation paths.
- Recording QA Lead decisions and producing audit evidence.
- Evaluating go-live readiness.

## Instructions

1. Create the sign-off table. Only the QA Lead role may insert:

        CREATE TABLE IF NOT EXISTS RECON_DB.CONTROL.RECON_SIGNOFF (
            SIGNOFF_ID   STRING DEFAULT UUID_STRING(),
            RUN_ID       STRING,
            CONFIG_ID    STRING,
            DECISION     STRING,              -- SIGNED_OFF, INVESTIGATE, REJECTED
            RATIONALE    STRING,
            REVIEWER     STRING DEFAULT CURRENT_USER(),
            DECIDED_AT   TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        GRANT INSERT, SELECT ON TABLE RECON_DB.CONTROL.RECON_SIGNOFF TO ROLE QA_LEAD_ROLE;
        GRANT SELECT ON TABLE RECON_DB.CONTROL.RECON_SIGNOFF TO ROLE RECON_AGENT_ROLE;

2. Publish the run summary view:

        CREATE OR REPLACE VIEW RECON_DB.REPORTING.V_RECON_SUMMARY AS
        SELECT r.RUN_ID, r.MILESTONE, c.SOURCE_SYSTEM, c.SOURCE_OBJECT, c.TARGET_OBJECT,
               c.CRITICALITY, r.STATUS AS RUN_STATUS,
               COUNT_IF(res.STATUS = 'FAIL') AS FAILED_CHECKS,
               COUNT_IF(res.STATUS = 'WARN') AS WARN_CHECKS,
               MAX_BY(s.DECISION, s.DECIDED_AT) AS LATEST_DECISION,
               r.ENDED_AT
        FROM RECON_DB.CONTROL.RECON_RUN r
        JOIN RECON_DB.CONTROL.RECON_CONFIG c ON c.CONFIG_ID = r.CONFIG_ID
        LEFT JOIN RECON_DB.CONTROL.RECON_RESULT res ON res.RUN_ID = r.RUN_ID
        LEFT JOIN RECON_DB.CONTROL.RECON_SIGNOFF s ON s.RUN_ID = r.RUN_ID
        GROUP BY ALL;

3. Publish the cutover readiness gate:

        CREATE OR REPLACE VIEW RECON_DB.REPORTING.V_CUTOVER_READINESS AS
        WITH LATEST AS (
            SELECT *
            FROM RECON_DB.REPORTING.V_RECON_SUMMARY
            WHERE MILESTONE IS NOT NULL
            QUALIFY ROW_NUMBER() OVER (PARTITION BY MILESTONE, TARGET_OBJECT ORDER BY ENDED_AT DESC) = 1
        )
        SELECT MILESTONE,
               COUNT(*) AS OBJECTS_IN_SCOPE,
               COUNT_IF(RUN_STATUS = 'PASS' OR LATEST_DECISION = 'SIGNED_OFF') AS OBJECTS_CLEARED,
               COUNT_IF(NOT (RUN_STATUS = 'PASS' OR LATEST_DECISION = 'SIGNED_OFF')) AS OBJECTS_BLOCKING,
               IFF(OBJECTS_BLOCKING = 0, 'READY', 'BLOCKED') AS GATE_STATUS
        FROM LATEST
        GROUP BY MILESTONE;

4. Report structure. Each reconciliation report contains, in order:
    - Executive summary: milestone, objects in scope, cleared, blocking, gate status.
    - Per-object status table: row count, control totals, key-field result, decision.
    - Variance detail: ranked FAIL findings with impact, root cause hint, and lineage path.
    - Evidence appendix: RUN_IDs, LOAD_IDs, query IDs, config versions, reviewer decisions.

5. Notify and escalate. Create a notification integration and alert on FAIL for HIGH criticality objects:

        CREATE NOTIFICATION INTEGRATION IF NOT EXISTS RECON_EMAIL_INT
            TYPE = EMAIL ENABLED = TRUE
            ALLOWED_RECIPIENTS = ('qa.lead@company.com');

        CREATE OR REPLACE ALERT RECON_DB.CONTROL.ALR_RECON_FAIL
            WAREHOUSE = RECON_WH
            SCHEDULE  = '15 MINUTE'
            IF (EXISTS (
                SELECT 1 FROM RECON_DB.REPORTING.V_RECON_SUMMARY
                WHERE RUN_STATUS = 'FAIL' AND CRITICALITY = 'HIGH'
                  AND LATEST_DECISION IS NULL
                  AND ENDED_AT >= SNOWFLAKE.ALERT.LAST_SUCCESSFUL_SCHEDULED_TIME()))
            THEN
                CALL SYSTEM$SEND_EMAIL('RECON_EMAIL_INT', 'qa.lead@company.com',
                    'Reconciliation FAIL: action required',
                    'One or more HIGH criticality objects failed reconciliation. Review RECON_DB.REPORTING.V_RECON_SUMMARY.');

        ALTER ALERT RECON_DB.CONTROL.ALR_RECON_FAIL RESUME;

    Escalate to the program sponsor when a FAIL remains without a decision for more than one business day before a milestone.

6. Cortex Agents coordination. Configure a Cortex Agent for the QA Lead with:
    - A Cortex Analyst tool over a semantic view on V_RECON_SUMMARY, RECON_RESULT, and RECON_VARIANCE, so the QA Lead can ask questions such as "which objects block GO_LIVE" or "total amount variance for SAP_ECC".
    - A tool that calls a procedure to enqueue a MANUAL re-run for a given CONFIG_ID.
    - No tool that writes to RECON_SIGNOFF; decisions are always recorded by the QA Lead directly.

7. Audit retention. Keep all control tables for the retention period the client's audit policy requires; Snowflake Horizon lineage and ACCESS_HISTORY provide the trace from report back to source loads.

## Coding Conventions
- The agent role has SELECT only on RECON_SIGNOFF; sign-off is technically impossible for the agent, not just discouraged.
- A SIGNED_OFF decision on a FAIL requires a non-empty RATIONALE; enforce this in the sign-off procedure or UI.
- Readiness is always computed from the latest run per object and milestone; never from manually maintained status lists.
- Notification bodies reference views and RUN_IDs only; never include record values or sensitive fields.
- Report numbers come from control tables, not from ad hoc queries, so every figure is reproducible for auditors.