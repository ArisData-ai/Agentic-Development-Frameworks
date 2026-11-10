---
name: ci-cd-release-gatekeeper-handoff
description: This skill defines how the CI/CD Agent hands promotion readiness to the Release Gatekeeper, generating a structured readiness report from control tables, sending notifications through a Snowflake email notification integration, and recording approve or reject decisions through a role-restricted stored procedure that enforces separation of duties. Use this skill for any task involving approval gates, release sign-off, building a promotion readiness summary, notifying approvers by email or Slack webhook, approving or blocking a QA or PROD release, or auditing who approved a deployment.
---

# CI/CD Release Gatekeeper Handoff

## Overview

The handoff is the boundary between automation and human accountability. The agent summarizes everything an approver needs into one readiness report: what changed, which checks passed or warned, drift findings, and risk. It then waits. Only the RELEASE_GATEKEEPER role can approve or reject, and the approver cannot be the author of the change.

## When to Use

- A PROMOTION_REQUESTS row is created with DECISION = 'PENDING'.
- Approvers need a consistent, readable release summary.
- An organization needs auditable sign-off for regulated deployments.
- An approval or rejection must be recorded or queried.

## Instructions

1. Build the readiness report as a VARIANT from control data.

        SELECT OBJECT_CONSTRUCT(
            'run_id',        r.RUN_ID,
            'commit',        r.COMMIT_HASH,
            'branch',        r.BRANCH,
            'blocking_failed', COUNT_IF(c.SEVERITY = 'BLOCKING' AND NOT c.PASSED),
            'warnings',      ARRAY_AGG(CASE WHEN c.SEVERITY = 'WARNING' AND NOT c.PASSED
                                            THEN OBJECT_CONSTRUCT('check', c.CHECK_NAME, 'detail', c.DETAIL) END),
            'checks_total',  COUNT(c.CHECK_NAME),
            'recommendation', IFF(COUNT_IF(c.SEVERITY = 'BLOCKING' AND NOT c.PASSED) = 0, 'READY', 'NOT_READY')
        ) AS READINESS
        FROM CICD_DB.CONTROL.PIPELINE_RUNS r
        LEFT JOIN CICD_DB.CONTROL.CHECK_RESULTS c ON c.RUN_ID = r.RUN_ID
        WHERE r.RUN_ID = :run_id
        GROUP BY r.RUN_ID, r.COMMIT_HASH, r.BRANCH;

2. Optionally add a plain-language summary with Cortex for approvers.

        SELECT AI_COMPLETE(
            'mistral-large2',
            'Summarize this release readiness report for a release approver in 5 bullet points, '
            || 'highlighting risks and warnings: ' || TO_VARCHAR(:readiness)
        ) AS APPROVER_SUMMARY;

3. Notify the Gatekeeper.

        CREATE OR REPLACE NOTIFICATION INTEGRATION CICD_EMAIL_INT
            TYPE = EMAIL
            ENABLED = TRUE
            ALLOWED_RECIPIENTS = ('release-approvers@<company>.com');

        CALL SYSTEM$SEND_EMAIL(
            'CICD_EMAIL_INT',
            'release-approvers@<company>.com',
            'Promotion pending: ' || :target_env || ' / ' || :commit_short,
            :approver_summary
        );

    For Slack or Teams, use a webhook notification integration with SYSTEM$SEND_SNOWFLAKE_NOTIFICATION.

4. Record decisions only through a role-restricted procedure.

        CREATE OR REPLACE PROCEDURE CICD_DB.ORCH.SP_DECIDE_PROMOTION(
            P_REQUEST_ID STRING, P_DECISION STRING, P_NOTE STRING)
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS CALLER
        AS
        $$
        DECLARE
            v_status STRING;
        BEGIN
            IF (NOT IS_ROLE_IN_SESSION('RELEASE_GATEKEEPER')) THEN
                RETURN 'DENIED: RELEASE_GATEKEEPER role required';
            END IF;
            IF (P_DECISION NOT IN ('APPROVED', 'REJECTED')) THEN
                RETURN 'INVALID_DECISION';
            END IF;
            SELECT DECISION INTO :v_status FROM CICD_DB.CONTROL.PROMOTION_REQUESTS WHERE REQUEST_ID = :P_REQUEST_ID;
            IF (v_status <> 'PENDING') THEN
                RETURN 'ALREADY_DECIDED: ' || v_status;
            END IF;
            UPDATE CICD_DB.CONTROL.PROMOTION_REQUESTS
                SET DECISION = :P_DECISION, DECIDED_BY = CURRENT_USER(),
                    DECISION_NOTE = :P_NOTE, DECIDED_AT = CURRENT_TIMESTAMP()
                WHERE REQUEST_ID = :P_REQUEST_ID;
            RETURN P_DECISION;
        END;
        $$;

        GRANT USAGE ON PROCEDURE CICD_DB.ORCH.SP_DECIDE_PROMOTION(STRING, STRING, STRING) TO ROLE RELEASE_GATEKEEPER;
        GRANT UPDATE ON TABLE CICD_DB.CONTROL.PROMOTION_REQUESTS TO ROLE RELEASE_GATEKEEPER;

5. Enforce separation of duties. Before executing an approved promotion, ci-cd-environment-promotion must verify that DECIDED_BY differs from the commit author captured in the run metadata.

6. Apply expiry. Mark PENDING requests older than the configured window, 72 hours by default, as REJECTED with DECISION_NOTE = 'EXPIRED', so stale releases never deploy.

## Coding Conventions

- The readiness report schema is stable: run_id, commit, branch, blocking_failed, warnings, checks_total, and recommendation.
- Never auto-approve protected environments, even when every check passes.
- Every decision row stores the approver, timestamp, and a mandatory note for rejections.
- Notifications contain summaries and links to control records, never credentials or raw sensitive data.