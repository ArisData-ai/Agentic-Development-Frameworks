---
name: deployment-validation-state-comparison-escalation
description: This skill defines how the Deployment Validation Agent turns check results into a scored go or no-go verdict, uses Snowflake Cortex AI and a Cortex Agent to explain differences between actual and expected state, and hands a structured evidence package to the Release Gatekeeper through an inbox table and email notifications. Use this skill for any task involving release verdicts, go or no-go decisions, rollback recommendations, Cortex Agent setup, AI-generated validation summaries, semantic views over validation results, Release Gatekeeper handoff, release alerts by email, Slack, or Microsoft Teams, or reporting on deployment quality.
---
# Deployment Validation State Comparison and Escalation

## Overview
This skill covers the agent's Hand off step. It reads all results for a run, applies deterministic severity rules to produce a verdict, uses Cortex AI to write a plain-language explanation, and publishes the evidence package to the Release Gatekeeper. A Cortex Agent sits on top of the results so release owners can ask follow-up questions. The verdict rules are deterministic; the AI only explains findings and never decides go or no-go.

## When to Use
- Producing the final verdict and evidence for a deployment.
- Setting up or tuning the Cortex Agent that answers release owners' questions.
- Configuring notifications and the Release Gatekeeper inbox.
- Building release quality reporting across deployments.

## Instructions

### 1. Release Gatekeeper inbox and notification integration
    CREATE TABLE IF NOT EXISTS DEPLOY_VALIDATION_DB.CORE.RELEASE_GATEKEEPER_INBOX (
        RUN_ID            STRING,
        DEPLOYMENT_ID     STRING,
        RELEASE_ID        STRING,
        ENVIRONMENT       STRING,
        VERDICT           STRING,
        RECOMMENDATION    STRING,     -- CONFIRM_RELEASE | REVIEW_WARNINGS | ROLLBACK
        SUMMARY           STRING,
        EVIDENCE          VARIANT,
        DECISION          STRING,     -- set by the Release Gatekeeper: CONFIRMED | ROLLED_BACK
        DECIDED_BY        STRING,
        DECIDED_AT        TIMESTAMP_LTZ,
        CREATED_AT        TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

    CREATE NOTIFICATION INTEGRATION IF NOT EXISTS VALIDATION_EMAIL_INT
        TYPE = EMAIL
        ENABLED = TRUE
        ALLOWED_RECIPIENTS = ('release-gatekeeper@company.com');

### 2. Verdict and escalation procedure
    CREATE OR REPLACE PROCEDURE DEPLOY_VALIDATION_DB.CORE.BUILD_VERDICT_AND_ESCALATE(
        P_DEPLOYMENT_ID STRING, P_RUN_ID STRING, P_START TIMESTAMP_LTZ)
    RETURNS VARIANT
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    $$
    DECLARE
        v_crit NUMBER; v_high NUMBER; v_warn NUMBER; v_err NUMBER;
        v_verdict STRING; v_reco STRING; v_summary STRING;
        v_evidence VARIANT; v_release STRING; v_env STRING;
    BEGIN
        SELECT
            COUNT_IF(STATUS = 'FAIL' AND SEVERITY = 'CRITICAL'),
            COUNT_IF(STATUS = 'FAIL' AND SEVERITY = 'HIGH'),
            COUNT_IF(STATUS = 'WARN' OR (STATUS = 'FAIL' AND SEVERITY IN ('MEDIUM', 'LOW'))),
            COUNT_IF(STATUS = 'ERROR'),
            ARRAY_AGG(OBJECT_CONSTRUCT('category', CHECK_CATEGORY, 'check', CHECK_NAME, 'object', OBJECT_NAME,
                      'severity', SEVERITY, 'status', STATUS, 'expected', EXPECTED, 'actual', ACTUAL,
                      'details', DETAILS)) WITHIN GROUP (ORDER BY SEVERITY, CHECK_CATEGORY)
        INTO :v_crit, :v_high, :v_warn, :v_err, :v_evidence
        FROM DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
        WHERE RUN_ID = :P_RUN_ID AND STATUS <> 'PASS';

        SELECT RELEASE_ID, ENVIRONMENT INTO :v_release, :v_env
        FROM DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_LOG WHERE DEPLOYMENT_ID = :P_DEPLOYMENT_ID;

        v_verdict := CASE WHEN v_crit + v_high + v_err > 0 THEN 'NO_GO'
                          WHEN v_warn > 0 THEN 'GO_WITH_WARNINGS'
                          ELSE 'GO' END;
        v_reco := CASE v_verdict WHEN 'NO_GO' THEN 'ROLLBACK'
                                 WHEN 'GO_WITH_WARNINGS' THEN 'REVIEW_WARNINGS'
                                 ELSE 'CONFIRM_RELEASE' END;

        v_summary := CASE WHEN v_verdict = 'GO'
            THEN 'All post-deployment checks passed.'
            ELSE SNOWFLAKE.CORTEX.COMPLETE('mistral-large2',
                'You are a Snowflake release validation assistant. Write at most 6 short sentences for a release owner. ' ||
                'State the verdict ' || :v_verdict || ', name each failed object and the difference between expected and actual state, ' ||
                'and list the most likely cause per finding. Do not invent findings. Findings JSON: ' ||
                LEFT(TO_JSON(:v_evidence), 60000))
        END;

        INSERT INTO DEPLOY_VALIDATION_DB.CORE.VALIDATION_VERDICTS
            (RUN_ID, DEPLOYMENT_ID, RELEASE_ID, ENVIRONMENT, VERDICT, CRITICAL_FAILS, HIGH_FAILS, WARNINGS,
             SUMMARY, EVIDENCE, VALIDATION_SECS)
        SELECT :P_RUN_ID, :P_DEPLOYMENT_ID, :v_release, :v_env, :v_verdict, :v_crit, :v_high, :v_warn,
               :v_summary, :v_evidence, DATEDIFF('second', :P_START, CURRENT_TIMESTAMP());

        INSERT INTO DEPLOY_VALIDATION_DB.CORE.RELEASE_GATEKEEPER_INBOX
            (RUN_ID, DEPLOYMENT_ID, RELEASE_ID, ENVIRONMENT, VERDICT, RECOMMENDATION, SUMMARY, EVIDENCE)
        SELECT :P_RUN_ID, :P_DEPLOYMENT_ID, :v_release, :v_env, :v_verdict, :v_reco, :v_summary, :v_evidence;

        IF (v_verdict <> 'GO') THEN
            CALL SYSTEM$SEND_EMAIL(
                'VALIDATION_EMAIL_INT',
                'release-gatekeeper@company.com',
                '[' || :v_verdict || '] Release ' || :v_release || ' in ' || :v_env,
                :v_summary || '\n\nCritical: ' || :v_crit || '  High: ' || :v_high || '  Warnings: ' || :v_warn ||
                '\nRecommendation: ' || :v_reco || '\nRun ID: ' || :P_RUN_ID);
        END IF;

        RETURN OBJECT_CONSTRUCT('run_id', P_RUN_ID, 'verdict', v_verdict, 'recommendation', v_reco,
                                'critical', v_crit, 'high', v_high, 'warnings', v_warn);
    END;
    $$;

### 3. Semantic view over validation results
    CREATE OR REPLACE SEMANTIC VIEW DEPLOY_VALIDATION_DB.CORE.SV_VALIDATION_RESULTS
        TABLES (
            results AS DEPLOY_VALIDATION_DB.CORE.VALIDATION_RESULTS
                COMMENT = 'One row per post-deployment check',
            verdicts AS DEPLOY_VALIDATION_DB.CORE.VALIDATION_VERDICTS PRIMARY KEY (RUN_ID)
                COMMENT = 'One verdict per validation run'
        )
        RELATIONSHIPS (
            results_to_verdicts AS results (RUN_ID) REFERENCES verdicts
        )
        DIMENSIONS (
            results.check_category AS results.CHECK_CATEGORY,
            results.check_name AS results.CHECK_NAME,
            results.object_name AS results.OBJECT_NAME,
            results.severity AS results.SEVERITY,
            results.status AS results.STATUS,
            verdicts.release_id AS verdicts.RELEASE_ID,
            verdicts.environment AS verdicts.ENVIRONMENT,
            verdicts.verdict AS verdicts.VERDICT,
            verdicts.created_at AS verdicts.CREATED_AT
        )
        METRICS (
            results.failed_checks AS COUNT_IF(results.STATUS = 'FAIL'),
            results.total_checks AS COUNT(results.CHECK_NAME),
            verdicts.no_go_runs AS COUNT_IF(verdicts.VERDICT = 'NO_GO'),
            verdicts.avg_validation_secs AS AVG(verdicts.VALIDATION_SECS)
        );

### 4. Cortex Agent for release owners
    CREATE OR REPLACE AGENT DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_VALIDATION_AGENT
        COMMENT = 'Explains post-deployment validation findings and supports Release Gatekeeper decisions'
        FROM SPECIFICATION
    $$
    models:
      orchestration: auto
    instructions:
      orchestration: >
        Answer questions about deployment validation runs, verdicts, and failed checks.
        Always cite RUN_ID, object names, and expected versus actual values.
        Never change a verdict and never perform a rollback; recommend actions only.
      response: >
        Be concise. Lead with the verdict, then list blocking findings by severity.
    tools:
      - tool_spec:
          type: cortex_analyst_text_to_sql
          name: validation_results
          description: Queries validation results, verdicts, and trends across releases and environments.
    tool_resources:
      validation_results:
        semantic_view: DEPLOY_VALIDATION_DB.CORE.SV_VALIDATION_RESULTS
        execution_environment:
          type: warehouse
          warehouse: VALIDATION_WH
    $$;

    GRANT USAGE ON AGENT DEPLOY_VALIDATION_DB.CORE.DEPLOYMENT_VALIDATION_AGENT TO ROLE RELEASE_OWNER_ROLE;

### 5. Release Gatekeeper decision write-back
    UPDATE DEPLOY_VALIDATION_DB.CORE.RELEASE_GATEKEEPER_INBOX
    SET DECISION = 'ROLLED_BACK', DECIDED_BY = CURRENT_USER(), DECIDED_AT = CURRENT_TIMESTAMP()
    WHERE RUN_ID = '<run_id>' AND DECISION IS NULL;

### 6. Release quality reporting
    SELECT ENVIRONMENT,
           DATE_TRUNC('week', CREATED_AT) AS WEEK,
           COUNT(*) AS RUNS,
           COUNT_IF(VERDICT = 'NO_GO') AS NO_GO_RUNS,
           ROUND(AVG(VALIDATION_SECS)) AS AVG_SECS_TO_VERDICT
    FROM DEPLOY_VALIDATION_DB.CORE.VALIDATION_VERDICTS
    GROUP BY 1, 2
    ORDER BY 2 DESC, 1;

## Best Practices
- Keep the verdict rules in SQL and under version control; the AI explains results, it does not decide them.
- Treat any `ERROR` as blocking, so a validation that could not finish never produces a false `GO`.
- Shorten the evidence passed to Cortex and tell the model not to invent findings; keep the full evidence in `VARIANT` columns.
- Route `NO_GO` alerts to an on-call channel; for Slack or Microsoft Teams, use a webhook notification integration with `SYSTEM$SEND_SNOWFLAKE_NOTIFICATION`.
- Track time from `COMPLETED_AT` to verdict as the main SLA metric, with a target of under 10 minutes.
- Choose a Cortex model that is available in the account's region, or enable cross-region inference.

## Coding Conventions
- Verdict values: `GO`, `GO_WITH_WARNINGS`, `NO_GO`. Recommendation values: `CONFIRM_RELEASE`, `REVIEW_WARNINGS`, `ROLLBACK`.
- Only the Release Gatekeeper updates the `DECISION` columns; the agent only inserts rows.
- The model name in Cortex calls is a single literal that is easy to find and change per environment.
- Agent instructions must forbid changing state, both in the orchestration and the response sections.