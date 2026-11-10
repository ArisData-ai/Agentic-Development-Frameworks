---
name: remediation-safe-action-execution
description: This skill defines safe remediation execution for the Remediation Agent, which runs only allowlisted, low-risk recovery actions such as task retries, task resumes, dynamic table refreshes, and stream recreation through a controlled stored procedure and Snowflake Tasks with retry limits and outcome logging. Use this skill for any task involving pre-approved remediation actions, automatic retry of failed tasks, EXECUTE TASK RETRY LAST, resuming suspended tasks, refreshing dynamic tables, building an action allowlist, enforcing execution guardrails, or recording remediation outcomes.
---
# Remediation Safe Action Execution

## Overview
Execution is the only component that changes production state. It runs SAFE tier actions automatically and APPROVED actions after an Operations Lead decision, always through one audited stored procedure owned by a restricted executor role.

## When to Use
- Adding or removing an action type from the safe allowlist
- Building or modifying the executor procedure
- Setting retry limits, cooldowns, or blast-radius guardrails

## Instructions
1. Maintain the allowlist as data, not code.

        CREATE TABLE IF NOT EXISTS OPS_DB.REMEDIATION.SAFE_ACTION_ALLOWLIST (
            action_type    STRING,
            sql_template   STRING,
            max_attempts   INT,
            cooldown_min   INT
        );

        INSERT INTO OPS_DB.REMEDIATION.SAFE_ACTION_ALLOWLIST VALUES
            ('RETRY_TASK',  'EXECUTE TASK {object} RETRY LAST', 2, 10),
            ('RUN_TASK',    'EXECUTE TASK {object}',            2, 10),
            ('RESUME_TASK', 'ALTER TASK {object} RESUME',       1, 30),
            ('REFRESH_DT',  'ALTER DYNAMIC TABLE {object} REFRESH', 2, 15);

2. Create the executor procedure; it rebuilds SQL from the template and never runs raw model output for SAFE actions.

        CREATE OR REPLACE PROCEDURE OPS_DB.REMEDIATION.EXECUTE_PROPOSAL(p_proposal_id STRING)
            RETURNS STRING
            LANGUAGE SQL
            EXECUTE AS OWNER
        AS
        $$
        DECLARE
            v_action STRING; v_object STRING; v_tier STRING; v_status STRING;
            v_sql STRING; v_attempts INT; v_max INT;
        BEGIN
            SELECT p.action_type, i.object_name, p.risk_tier, p.status
              INTO :v_action, :v_object, :v_tier, :v_status
              FROM OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS p
              JOIN OPS_DB.REMEDIATION.INCIDENTS i ON i.incident_id = p.incident_id
             WHERE p.proposal_id = :p_proposal_id;

            IF (NOT (v_tier = 'SAFE' OR v_status = 'APPROVED')) THEN
                RETURN 'REJECTED: not safe and not approved';
            END IF;

            SELECT REPLACE(sql_template, '{object}', :v_object), max_attempts
              INTO :v_sql, :v_max
              FROM OPS_DB.REMEDIATION.SAFE_ACTION_ALLOWLIST
             WHERE action_type = :v_action;

            SELECT COUNT(*) INTO :v_attempts
              FROM OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS p
              JOIN OPS_DB.REMEDIATION.INCIDENTS i ON i.incident_id = p.incident_id
             WHERE i.object_name = :v_object AND p.executed_at > DATEADD('hour', -1, CURRENT_TIMESTAMP());

            IF (v_attempts >= v_max) THEN
                UPDATE OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS
                   SET risk_tier = 'REVIEW', status = 'PENDING'
                 WHERE proposal_id = :p_proposal_id;
                RETURN 'ESCALATED: retry limit reached';
            END IF;

            EXECUTE IMMEDIATE :v_sql;

            UPDATE OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS
               SET status = 'EXECUTED', executed_at = CURRENT_TIMESTAMP()
             WHERE proposal_id = :p_proposal_id;
            RETURN 'EXECUTED: ' || v_sql;
        EXCEPTION
            WHEN OTHER THEN
                UPDATE OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS
                   SET status = 'FAILED', outcome = :SQLERRM
                 WHERE proposal_id = :p_proposal_id;
                RETURN 'FAILED: ' || SQLERRM;
        END;
        $$;

3. Drive execution with a Stream on proposals and a triggered Task.

        CREATE OR REPLACE STREAM OPS_DB.REMEDIATION.PROPOSALS_STREAM
            ON TABLE OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS;

        CREATE OR REPLACE TASK OPS_DB.REMEDIATION.TASK_EXECUTE_READY
            WAREHOUSE = REMEDIATION_WH
            WHEN SYSTEM$STREAM_HAS_DATA('OPS_DB.REMEDIATION.PROPOSALS_STREAM')
        AS
            EXECUTE IMMEDIATE $$
            DECLARE c CURSOR FOR
                SELECT proposal_id FROM OPS_DB.REMEDIATION.PROPOSALS_STREAM
                WHERE (risk_tier = 'SAFE' AND status = 'PENDING') OR status = 'APPROVED';
            BEGIN
                FOR r IN c DO
                    CALL OPS_DB.REMEDIATION.EXECUTE_PROPOSAL(r.proposal_id);
                END FOR;
            END;
            $$;

        ALTER TASK OPS_DB.REMEDIATION.TASK_EXECUTE_READY RESUME;

4. Verify recovery after execution by re-checking the original signal; set outcome to RESOLVED or NOT_RESOLVED.
5. Grant OPERATE on target tasks and dynamic tables to REMEDIATION_EXECUTOR_ROLE only.

## Coding Conventions
- Execute SAFE actions only from allowlist templates; APPROVED CUSTOM_SQL runs only after Operations Lead sign-off.
- Never allow DROP, TRUNCATE, DELETE, GRANT, or REVOKE through the executor.
- Enforce max attempts and cooldowns per object to prevent retry storms.
- Capture SQLERRM and the executed statement for every run.