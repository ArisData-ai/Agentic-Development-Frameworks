---
name: release-gatekeeper-handoff
description: This skill defines how the Release Agent packages a release for the Release Gatekeeper, submits it for review, and processes approve, defer, or modify decisions through a governed state machine, decision tables, notifications, and rework loops back into scope assembly. Use this skill for any task involving release approval workflows, go-live sign-off, Change Advisory Board submissions, ServiceNow change requests, deferral handling, release modification requests, approval notifications through Slack, Teams, or email, or handing approved packages to the CI/CD and deployment stages.
---
# Release Gatekeeper Handoff

## Overview
The handoff is the human-in-the-loop control point. The Release Agent submits a complete package; the Release Gatekeeper (release manager or Change Advisory Board) approves, defers, or modifies it. The agent never approves its own work. Approved packages flow to the CI/CD Agent; deferred and modified packages loop back.

## When to Use
- Submitting a READY package for go-live review.
- Recording and acting on gatekeeper decisions.
- Sending submission and decision notifications.
- Syncing with external change systems such as ServiceNow or Jira.
- Releasing an approved package to the CI/CD stage.

## Instructions

1. Check submission readiness. A package is submittable only when all of these hold:
    - STATUS = 'READY'
    - DEPENDENCY_REPORT:blocking is empty
    - RELEASE_NOTES contains BUSINESS, ENGINEERING, and COMPLIANCE keys
    - Audit log contains SCOPE_ASSEMBLED, DEPS_CHECKED, and NOTES_DRAFTED events
    - PLANNED_WINDOW is set

2. Create the decision table.

    CREATE TABLE IF NOT EXISTS RELEASE_DB.CORE.GATEKEEPER_DECISIONS (
        DECISION_ID     STRING DEFAULT UUID_STRING(),
        RELEASE_ID      STRING NOT NULL,
        DECISION        STRING NOT NULL,   -- APPROVE | DEFER | MODIFY
        REASON          STRING,
        DEFER_UNTIL     TIMESTAMP_LTZ,
        MODIFICATIONS   VARIANT,           -- {"remove_changes": [...], "notes_feedback": "..."}
        EXTERNAL_REF    STRING,            -- ServiceNow CHG number
        DECIDED_BY      STRING DEFAULT CURRENT_USER(),
        DECIDED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    GRANT INSERT, SELECT ON TABLE RELEASE_DB.CORE.GATEKEEPER_DECISIONS TO ROLE RELEASE_GATEKEEPER_ROLE;
    GRANT SELECT ON TABLE RELEASE_DB.CORE.GATEKEEPER_DECISIONS TO ROLE RELEASE_AGENT_ROLE;

3. Submit and notify.

    CREATE OR REPLACE PROCEDURE RELEASE_DB.CORE.SP_SUBMIT_TO_GATEKEEPER(REL_ID STRING)
    RETURNS STRING
    LANGUAGE SQL
    AS
    $$
    BEGIN
        UPDATE RELEASE_DB.CORE.RELEASE_PACKAGES
        SET STATUS = 'SUBMITTED', UPDATED_AT = CURRENT_TIMESTAMP()
        WHERE RELEASE_ID = :REL_ID AND STATUS = 'READY';

        CALL RELEASE_DB.AUDIT.SP_LOG_RELEASE_EVENT(:REL_ID, 'SUBMITTED', 'READY', 'SUBMITTED', 'AGENT', NULL);

        CALL SYSTEM$SEND_SNOWFLAKE_NOTIFICATION(
            SNOWFLAKE.NOTIFICATION.TEXT_PLAIN(
                'Release ' || :REL_ID || ' is ready for review. Open the release review app to approve, defer, or modify.'),
            SNOWFLAKE.NOTIFICATION.INTEGRATION('RELEASE_SLACK_INT'));
        RETURN 'SUBMITTED';
    END;
    $$;

    Create notification integrations for Slack or Teams webhooks and email as required by the client.

4. Process decisions with a stream and task.

    CREATE OR REPLACE STREAM RELEASE_DB.CORE.DECISIONS_STREAM
        ON TABLE RELEASE_DB.CORE.GATEKEEPER_DECISIONS APPEND_ONLY = TRUE;

    CREATE OR REPLACE TASK RELEASE_DB.CORE.PROCESS_DECISIONS
        WAREHOUSE = RELEASE_WH
        SCHEDULE = '5 MINUTE'
        WHEN SYSTEM$STREAM_HAS_DATA('RELEASE_DB.CORE.DECISIONS_STREAM')
    AS
        CALL RELEASE_DB.CORE.SP_APPLY_DECISIONS();

    SP_APPLY_DECISIONS behavior:
    - APPROVE: set STATUS = 'APPROVED', assign RELEASE_VERSION, log APPROVED with ACTOR_TYPE = 'HUMAN', trigger the CI/CD handoff.
    - DEFER: set STATUS = 'DEFERRED', store DEFER_UNTIL, log DEFERRED; a scheduled check returns it to READY after DEFER_UNTIL with a fresh dependency check.
    - MODIFY: set STATUS = 'MODIFY_REQUESTED', release removed changes (RELEASE_ID = NULL), pass notes_feedback to the notes regeneration step, then rerun scope, dependency, and notes steps, ending at READY.

5. Only accept decisions from RELEASE_GATEKEEPER_ROLE and reject decisions on packages not in SUBMITTED status. Validate in SP_APPLY_DECISIONS by joining to the audit log and current status.

6. Hand off approved packages to CI/CD with a deployable artifact.

    CREATE OR REPLACE VIEW RELEASE_DB.CORE.V_APPROVED_FOR_DEPLOY AS
    SELECT RELEASE_ID, RELEASE_VERSION, TARGET_ENV, PLANNED_WINDOW,
           DEPENDENCY_REPORT:deploy_order AS DEPLOY_ORDER, MANIFEST
    FROM RELEASE_DB.CORE.RELEASE_PACKAGES
    WHERE STATUS = 'APPROVED';

    The CI/CD Agent consumes this view, and the Deployment Validation Agent writes RELEASED once checks pass.

7. Optional external sync: call ServiceNow or Jira through an external access integration from a Python procedure, storing the returned change number in EXTERNAL_REF.

Best practices:
- Present the gatekeeper with one screen: summary, BUSINESS notes, blocking or impacted items, test evidence, window. A Streamlit in Snowflake review app works well.
- Require a REASON for DEFER and MODIFY.
- Re-run dependency checks on any package that waits more than 24 hours before approval.
- Never auto-approve, even for low-risk releases, unless the client's change policy explicitly defines a standard change category.

Output expectations: a SUBMITTED package with notification sent, recorded gatekeeper decisions, correct status transitions with audit events, and approved packages visible to CI/CD in deployment order.

## Coding Conventions
- DECISION values: APPROVE, DEFER, MODIFY only.
- MODIFICATIONS JSON keys: remove_changes, add_changes, notes_feedback, window_change.
- Notification integrations named RELEASE_<CHANNEL>_INT.
- All decision handling in one procedure, SP_APPLY_DECISIONS, for one auditable code path.
- Status updates always include a WHERE clause on the expected current status.