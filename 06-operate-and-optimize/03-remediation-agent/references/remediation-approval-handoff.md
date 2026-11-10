---
name: remediation-approval-handoff
description: This skill defines the Operations Lead approval handoff for the Remediation Agent, which notifies approvers of REVIEW tier proposals with diagnosis evidence and records approve, modify, or block decisions through stored procedures, notification integrations, and a Streamlit in Snowflake review console. Use this skill for any task involving human-in-the-loop approval, on-call escalation, sending remediation alerts to email, Slack, Microsoft Teams, or PagerDuty, building an approval queue UI, modifying proposed fixes, blocking unsafe actions, or auditing remediation decisions.
---
# Remediation Approval Handoff

## Overview
Handoff keeps a human in control of anything beyond pre-approved safe actions. The Operations Lead receives a concise diagnosis, reviews evidence, and approves, modifies, or blocks the proposal; every decision is audited.

## When to Use
- Configuring notifications for new REVIEW proposals
- Building the approval queue or decision procedures
- Defining escalation timers and SLA-based routing

## Instructions
1. Create a notification integration and send a structured message for each new REVIEW proposal.

        CREATE NOTIFICATION INTEGRATION IF NOT EXISTS REMEDIATION_EMAIL_INT
            TYPE = EMAIL
            ENABLED = TRUE
            ALLOWED_RECIPIENTS = ('ops-lead@company.com');

        CALL SYSTEM$SEND_EMAIL(
            'REMEDIATION_EMAIL_INT',
            'ops-lead@company.com',
            'SEV2 remediation pending: PROD_DB.ETL.LOAD_ORDERS',
            'Root cause: RESOURCE (warehouse suspended). Confidence 0.72. Proposed: ALTER WAREHOUSE ETL_WH RESUME. Review in console.'
        );

2. Route to Slack, Teams, or PagerDuty with a webhook notification integration when the team does not use email.
3. Expose decisions through a single procedure so the UI and any chat tool share one audit path.

        CREATE OR REPLACE PROCEDURE OPS_DB.REMEDIATION.DECIDE_PROPOSAL(
            p_proposal_id STRING, p_decision STRING, p_modified_sql STRING)
            RETURNS STRING
            LANGUAGE SQL
            EXECUTE AS CALLER
        AS
        $$
        BEGIN
            IF (p_decision NOT IN ('APPROVED', 'MODIFIED', 'BLOCKED')) THEN
                RETURN 'INVALID DECISION';
            END IF;
            UPDATE OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS
               SET status     = IFF(:p_decision = 'MODIFIED', 'APPROVED', :p_decision),
                   action_sql = COALESCE(:p_modified_sql, action_sql),
                   decided_by = CURRENT_USER(),
                   decided_at = CURRENT_TIMESTAMP()
             WHERE proposal_id = :p_proposal_id AND status = 'PENDING';
            RETURN 'RECORDED: ' || p_decision;
        END;
        $$;

        GRANT USAGE ON PROCEDURE OPS_DB.REMEDIATION.DECIDE_PROPOSAL(STRING, STRING, STRING)
            TO ROLE OPERATIONS_LEAD_ROLE;

4. Build a Streamlit in Snowflake console listing PENDING proposals with root cause, confidence, evidence links, and action SQL, plus Approve, Modify, and Block buttons that call DECIDE_PROPOSAL.
5. Escalate proposals left PENDING beyond the severity SLA (for SEV1, 15 minutes) to a secondary approver.
6. Feed BLOCKED decisions with the approver's reason back to the knowledge corpus as negative guidance.

## Coding Conventions
- Notifications state severity, object, root cause, confidence, and proposed action in under five lines.
- Only OPERATIONS_LEAD_ROLE may call DECIDE_PROPOSAL; the agent role never self-approves.
- Record decided_by and decided_at on every decision; never update decisions in place after execution.
- Modified SQL still passes the BLOCKED keyword check before execution.