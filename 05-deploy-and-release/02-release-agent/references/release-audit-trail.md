---
name: release-audit-trail
description: This skill defines how the Release Agent records an immutable, queryable audit trail for every release, capturing scope, approvals, evidence, lineage snapshots, and actor identity in append-only tables governed by Snowflake Horizon tags, masking policies, and ACCESS_HISTORY. Use this skill for any task involving release audit logs, change-management evidence, SOX, HIPAA, or ITIL compliance records, lineage capture, retention policies, audit queries, or preparing release records for compliance teams and external auditors.
---
# Release Audit Trail

## Overview
The audit trail makes every release traceable from ticket to commit to test run to approval to deployment. Records are append-only, tagged and retained through Snowflake Horizon, and readable by compliance teams without engineering involvement.

## When to Use
- Logging any release state transition or agent action.
- Capturing lineage and dependency snapshots at submission time.
- Answering auditor questions on who approved what and when.
- Configuring retention, tagging, and access controls for release records.

## Instructions

1. Create the append-only audit log.

    CREATE TABLE IF NOT EXISTS RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG (
        EVENT_ID       STRING DEFAULT UUID_STRING(),
        RELEASE_ID     STRING NOT NULL,
        EVENT_TYPE     STRING NOT NULL,   -- SCOPE_ASSEMBLED | DEPS_CHECKED | NOTES_DRAFTED | NOTES_VALIDATION_FAILED | SUBMITTED | APPROVED | DEFERRED | MODIFY_REQUESTED | RELEASED
        FROM_STATUS    STRING,
        TO_STATUS      STRING,
        ACTOR          STRING DEFAULT CURRENT_USER(),
        ACTOR_ROLE     STRING DEFAULT CURRENT_ROLE(),
        ACTOR_TYPE     STRING,            -- AGENT | HUMAN
        QUERY_ID       STRING DEFAULT LAST_QUERY_ID(),
        DETAILS        VARIANT,
        EVENT_TS       TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    )
    DATA_RETENTION_TIME_IN_DAYS = 90
    CHANGE_TRACKING = TRUE;

2. Enforce immutability: grant INSERT only, never UPDATE or DELETE, to every role except a break-glass owner.

    REVOKE UPDATE, DELETE, TRUNCATE ON TABLE RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG FROM ROLE RELEASE_AGENT_ROLE;
    GRANT INSERT ON TABLE RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG TO ROLE RELEASE_AGENT_ROLE;
    GRANT INSERT ON TABLE RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG TO ROLE RELEASE_GATEKEEPER_ROLE;

    For regulated workloads, add a backup set with a retention lock for WORM-style protection, or replicate the AUDIT schema to a separate account.

3. Provide a single logging procedure used by all release procedures.

    CREATE OR REPLACE PROCEDURE RELEASE_DB.AUDIT.SP_LOG_RELEASE_EVENT(
        REL_ID STRING, EVENT_TYPE STRING, FROM_STATUS STRING, TO_STATUS STRING, ACTOR_TYPE STRING, DETAILS VARIANT)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    BEGIN
        INSERT INTO RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG
            (RELEASE_ID, EVENT_TYPE, FROM_STATUS, TO_STATUS, ACTOR_TYPE, DETAILS)
        SELECT :REL_ID, :EVENT_TYPE, :FROM_STATUS, :TO_STATUS, :ACTOR_TYPE, :DETAILS;
        RETURN 'LOGGED';
    END;
    $$;

    Use EXECUTE AS CALLER so ACTOR and ACTOR_ROLE reflect the real caller.

4. Snapshot lineage and dependencies at submission time so evidence survives later object changes.

    CREATE TABLE IF NOT EXISTS RELEASE_DB.AUDIT.RELEASE_LINEAGE_SNAPSHOT (
        RELEASE_ID   STRING,
        OBJECT_FQN   STRING,
        DIRECTION    STRING,
        LINEAGE      VARIANT,
        CAPTURED_AT  TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

5. Apply Horizon governance tags.

    CREATE TAG IF NOT EXISTS RELEASE_DB.AUDIT.DATA_CLASS ALLOWED_VALUES 'AUDIT_RECORD', 'CHANGE_EVIDENCE';
    CREATE TAG IF NOT EXISTS RELEASE_DB.AUDIT.RETENTION_POLICY ALLOWED_VALUES '1Y', '7Y';

    ALTER TABLE RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG SET TAG
        RELEASE_DB.AUDIT.DATA_CLASS = 'AUDIT_RECORD',
        RELEASE_DB.AUDIT.RETENTION_POLICY = '7Y';

    Flag in-scope objects carrying sensitive tags (PII, PHI) in DETAILS so compliance notes surface them:

    SELECT OBJECT_DATABASE || '.' || OBJECT_SCHEMA || '.' || OBJECT_NAME AS OBJECT_FQN, TAG_NAME, TAG_VALUE
    FROM SNOWFLAKE.ACCOUNT_USAGE.TAG_REFERENCES
    WHERE TAG_NAME IN ('PII', 'PHI', 'SENSITIVITY');

6. Correlate with platform evidence using ACCESS_HISTORY and QUERY_HISTORY through QUERY_ID.

    SELECT a.RELEASE_ID, a.EVENT_TYPE, q.QUERY_TEXT, q.START_TIME, q.EXECUTION_STATUS
    FROM RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG a
    JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q ON a.QUERY_ID = q.QUERY_ID
    WHERE a.RELEASE_ID = :rel_id
    ORDER BY a.EVENT_TS;

7. Publish an auditor view.

    CREATE OR REPLACE SECURE VIEW RELEASE_DB.AUDIT.V_RELEASE_EVIDENCE AS
    SELECT p.RELEASE_ID, p.RELEASE_VERSION, p.TARGET_ENV, p.STATUS,
           f.value:change_id::STRING AS CHANGE_ID,
           f.value:ticket::STRING AS TICKET_ID,
           f.value:test_run::STRING AS TEST_RUN_ID,
           MAX(IFF(a.EVENT_TYPE = 'APPROVED', a.ACTOR, NULL)) AS APPROVED_BY,
           MAX(IFF(a.EVENT_TYPE = 'APPROVED', a.EVENT_TS, NULL)) AS APPROVED_AT
    FROM RELEASE_DB.CORE.RELEASE_PACKAGES p,
         LATERAL FLATTEN(p.MANIFEST:changes) f
    LEFT JOIN RELEASE_DB.AUDIT.RELEASE_AUDIT_LOG a ON a.RELEASE_ID = p.RELEASE_ID
    GROUP BY 1,2,3,4,5,6,7;

    GRANT SELECT ON VIEW RELEASE_DB.AUDIT.V_RELEASE_EVIDENCE TO ROLE RELEASE_AUDITOR_ROLE;

Best practices:
- Log before and after every state change; a missing event is an audit finding.
- Record the model name and template version used for notes generation in DETAILS.
- Never store secrets or raw sensitive data values in DETAILS; store object names and tags only.
- Separate duties: the agent role cannot write APPROVED events.

Output expectations: a complete, ordered event history per RELEASE_ID, a lineage snapshot per in-scope object, and a secure evidence view that compliance teams can query directly.

## Coding Conventions
- EVENT_TYPE values in UPPER_SNAKE_CASE from the controlled list only.
- All timestamps TIMESTAMP_LTZ; all DETAILS keys snake_case.
- Audit objects live only in RELEASE_DB.AUDIT.
- Auditor access only through SECURE views.
- ACTOR_TYPE must be set explicitly on every insert.