---
name: dq-qa-lead-escalation
description: This skill defines how the Data Quality Agent packages check results, quarantine counts, volume and freshness anomalies, and proposed rules into an evidence-backed escalation, notifies the QA Lead, and records approve, reject, release, or hold decisions in an audit trail, built on Snowflake Tasks, notification integrations, Cortex Agents, and Horizon access controls. Use this skill for any task involving DQ alerting, QA Lead approval workflows, human-in-the-loop sign-off, rule change approval, release or hold decisions for affected data, email or Slack notifications, or producing data quality audit evidence for finance, retail, and compliance teams.
---
# DQ QA Lead Escalation

## Overview
Escalation is the hand-off point. The agent presents what it found and what it did; the QA Lead owns every quality decision. Escalations are evidence-first, ranked by severity, and every decision is recorded with who, what, and when.

## When to Use
- A BLOCKER or MAJOR rule fails in a run
- Records or loads are held in quarantine
- New rules are PROPOSED and need approval
- Producing audit evidence of quality decisions

## Instructions

### 1. Create the escalation table
    CREATE TABLE IF NOT EXISTS DQ_DB.RESULTS.DQ_ESCALATIONS (
        ESCALATION_ID    STRING DEFAULT UUID_STRING(),
        RUN_ID           STRING,
        TABLE_FQN        STRING,
        PRIORITY         STRING,          -- P1 (BLOCKER) | P2 (MAJOR) | P3 (MINOR or proposals only)
        SUMMARY          STRING,
        EVIDENCE         VARIANT,         -- failed rules, metric values, quarantine counts, anomalies, proposals
        DECISION_REQUIRED ARRAY,          -- RELEASE_OR_HOLD | APPROVE_RULES
        STATUS           STRING DEFAULT 'PENDING',  -- PENDING | DECIDED | EXPIRED
        DECISION         VARIANT,
        DECIDED_BY       STRING,
        DECIDED_AT       TIMESTAMP_LTZ,
        CREATED_AT       TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

### 2. Build the evidence package
SP_ESCALATE_RUN(RUN_ID) assembles EVIDENCE as:

    {
      "run_id": "...", "table": "RAW_DB.ERP.SALES_ORDERS",
      "pass_rate_pct": 92.5,
      "failed_rules": [{"rule_id": "...", "dimension": "COMPLETENESS", "column": "CUSTOMER_ID",
                        "metric": 412, "threshold": "= 0", "severity": "MAJOR"}],
      "quarantined_rows": 412, "load_held": false,
      "anomalies": [{"type": "VOLUME", "rows": 18200, "baseline_mean": 41000}],
      "proposed_rules": [{"rule_id": "...", "dimension": "UNIQUENESS", "rationale": "..."}],
      "recommended_action": "Hold 412 rows; publish remainder; approve 2 proposed rules"
    }

Set PRIORITY from the highest failed severity. Skip escalation when all rules pass and no proposals exist; set DQ_RUN_LOG.STATUS = 'CLOSED' instead.

### 3. Notify the QA Lead
    CREATE NOTIFICATION INTEGRATION IF NOT EXISTS DQ_EMAIL_INT
        TYPE = EMAIL ENABLED = TRUE
        ALLOWED_RECIPIENTS = ('qa.lead@company.com');

    CALL SYSTEM$SEND_EMAIL(
        'DQ_EMAIL_INT',
        'qa.lead@company.com',
        'P2 Data Quality: RAW_DB.ERP.SALES_ORDERS, 412 rows held',
        'Run 7f3a...: 1 MAJOR rule failed (CUSTOMER_ID nulls = 412). 2 rules proposed. Review in DQ_DB.RESULTS.DQ_ESCALATIONS.');

For Slack or Teams, use a webhook notification integration with SYSTEM$SEND_SNOWFLAKE_NOTIFICATION. P1 escalations notify immediately; P3 escalations batch into a daily digest task.

### 4. Record QA Lead decisions
    CREATE OR REPLACE PROCEDURE DQ_DB.OPS.SP_RECORD_DECISION(ESCALATION_ID STRING, DECISION VARIANT)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    BEGIN
        IF (NOT IS_ROLE_IN_SESSION('DQ_QA_LEAD')) THEN
            RETURN 'DENIED: QA Lead role required';
        END IF;
        UPDATE DQ_DB.RESULTS.DQ_ESCALATIONS
           SET STATUS = 'DECIDED', DECISION = :DECISION,
               DECIDED_BY = CURRENT_USER(), DECIDED_AT = CURRENT_TIMESTAMP()
         WHERE ESCALATION_ID = :ESCALATION_ID AND STATUS = 'PENDING';
        RETURN 'OK';
    END;
    $$;

DECISION shape:

    {"data": "HOLD" | "RELEASE" | "DISCARD",
     "rules": [{"rule_id": "...", "action": "APPROVE" | "REJECT" | "MODIFY",
                "threshold_low": 0, "threshold_high": 0, "severity": "MAJOR"}],
     "note": "Source fix ticket DQ-123 raised"}

A follow-up task applies the decision: calls SP_RELEASE_QUARANTINE for data actions and SP_APPROVE_RULE for rule actions (sets STATUS = 'ACTIVE', APPROVED_BY, APPROVED_AT, and attaches the DMF).

### 5. Track SLAs and expiry
    P1: acknowledge within 1 hour, re-notify every hour until decided
    P2: decide within 1 business day
    P3: decide within 5 business days; expire unanswered proposals to REJECTED with note 'expired'
Held data stays HELD on expiry; it is never auto-released.

### 6. Report for stakeholders
Expose a view for analysts and auditors:

    CREATE OR REPLACE VIEW DQ_DB.RESULTS.V_DQ_DECISION_AUDIT AS
    SELECT e.TABLE_FQN, e.PRIORITY, e.SUMMARY, e.STATUS, e.DECISION:data::STRING AS DATA_DECISION,
           e.DECIDED_BY, e.DECIDED_AT, DATEDIFF('minute', e.CREATED_AT, e.DECIDED_AT) AS MINUTES_TO_DECISION
    FROM DQ_DB.RESULTS.DQ_ESCALATIONS e;

Cortex Analyst can sit on this view through a semantic model so stakeholders ask questions like "which tables had the most held rows last month".

## Coding Conventions
- One escalation per run; append new evidence to the same record rather than opening duplicates.
- Summaries state table, priority, counts, and the decision needed in one line; evidence lives in VARIANT.
- Decision procedures use EXECUTE AS CALLER with an explicit QA Lead role check; the agent role cannot call them.
- Never include unmasked sensitive values in notifications; reference record keys and counts only.
- Notification recipients come from DQ_DB.CONFIG.MONITORED_TABLES owners plus the QA Lead, never hard-coded in procedures.