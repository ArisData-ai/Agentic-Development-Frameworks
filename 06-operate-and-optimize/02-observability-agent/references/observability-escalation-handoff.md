---
name: observability-escalation-handoff
description: This skill defines how the Observability Agent escalates prioritized incidents to the Operations Lead through Snowflake Alerts, notification integrations and structured escalation payloads, then records acknowledgment, confirmed priority, assignee and suppression state for each incident. Use this skill for any task involving incident escalation, on-call notification, email, Slack, Teams or webhook alerts, Operations Lead handoff, acknowledgment tracking, priority confirmation, response assignment, maintenance window suppression, or human-in-the-loop incident routing.
---
# Observability Escalation Handoff

## Overview
This skill implements the Hand off stage. The agent sends a triaged view of each incident to the Operations Lead, who confirms priority and assigns response. The agent never assigns work or changes pipelines on its own. Routing follows priority:

    Priority   Channel                          Escalate within   Re-notify if unacknowledged
    --------   ------------------------------   ---------------   ---------------------------
    P1         Webhook (Slack/Teams/PagerDuty)  5 minutes         Every 15 minutes
               plus email
    P2         Webhook plus email               15 minutes        Every 60 minutes
    P3         Email digest                     Hourly            Next digest
    P4         Daily digest only                Daily             None

## When to Use
- Connecting the incident queue to email, Slack, Microsoft Teams or PagerDuty.
- Defining acknowledgment and assignment workflow for the Operations Lead.
- Suppressing notifications during planned maintenance or deployments.
- Auditing who confirmed and owned each incident.

## Instructions
1. Create notification integrations.

    CREATE OR REPLACE NOTIFICATION INTEGRATION OBS_EMAIL_INT
        TYPE = EMAIL ENABLED = TRUE
        ALLOWED_RECIPIENTS = ('ops-lead@company.com', 'data-ops@company.com');

    CREATE OR REPLACE SECRET OBS_DB.MONITOR.SLACK_WEBHOOK_SECRET
        TYPE = GENERIC_STRING SECRET_STRING = '<webhook-path>';

    CREATE OR REPLACE NOTIFICATION INTEGRATION OBS_SLACK_INT
        TYPE = WEBHOOK ENABLED = TRUE
        WEBHOOK_URL = 'https://hooks.slack.com/services/SNOWFLAKE_WEBHOOK_SECRET'
        WEBHOOK_SECRET = OBS_DB.MONITOR.SLACK_WEBHOOK_SECRET
        WEBHOOK_BODY_TEMPLATE = '{"text": "SNOWFLAKE_WEBHOOK_MESSAGE"}'
        WEBHOOK_HEADERS = ('Content-Type' = 'application/json');

2. Create the escalation log.

    CREATE TABLE IF NOT EXISTS OBS_DB.MONITOR.ESCALATIONS (
        ESCALATION_ID       STRING DEFAULT UUID_STRING(),
        INCIDENT_ID         STRING,
        AGENT_PRIORITY      STRING,
        CONFIRMED_PRIORITY  STRING,
        CHANNEL             STRING,
        SENT_AT             TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
        ACKNOWLEDGED_BY     STRING,
        ACKNOWLEDGED_AT     TIMESTAMP_LTZ,
        ASSIGNED_TO         STRING,
        NOTES               STRING
    );

    CREATE TABLE IF NOT EXISTS OBS_DB.MONITOR.SUPPRESSION_WINDOWS (
        PIPELINE_ID STRING, START_AT TIMESTAMP_LTZ, END_AT TIMESTAMP_LTZ, REASON STRING, CREATED_BY STRING
    );

3. Build a standard escalation payload.

    CREATE OR REPLACE VIEW OBS_DB.MONITOR.V_ESCALATION_PAYLOAD AS
    SELECT
        i.INCIDENT_ID,
        i.PRIORITY,
        '[' || i.PRIORITY || '] ' || r.DATA_PRODUCT || ' | ' || i.ROOT_PIPELINE_ID AS SUBJECT,
        'Impact score: ' || i.IMPACT_SCORE
          || '\nSignals: ' || i.SIGNAL_COUNT || ' (' || ARRAY_TO_STRING(i.SIGNAL_TYPES, ', ') || ')'
          || '\nAffected pipelines: ' || ARRAY_TO_STRING(i.AFFECTED_PIPELINES, ', ')
          || '\nFirst seen: ' || i.FIRST_SIGNAL_AT
          || '\nOwner: ' || r.OWNER_EMAIL
          || '\n\nTriage:\n' || COALESCE(i.TRIAGE_SUMMARY, 'Pending')
          || '\n\nConfirm: CALL OBS_DB.MONITOR.SP_ACK_INCIDENT(''' || i.INCIDENT_ID || ''', ''<P1-P4>'', ''<assignee>'');' AS BODY
    FROM OBS_DB.MONITOR.INCIDENTS i
    JOIN OBS_DB.MONITOR.PIPELINE_REGISTRY r ON r.PIPELINE_ID = i.ROOT_PIPELINE_ID
    WHERE i.STATUS = 'OPEN'
      AND NOT EXISTS (SELECT 1 FROM OBS_DB.MONITOR.SUPPRESSION_WINDOWS w
                      WHERE w.PIPELINE_ID = i.ROOT_PIPELINE_ID
                        AND CURRENT_TIMESTAMP() BETWEEN w.START_AT AND w.END_AT);

4. Send P1 and P2 incidents with a Snowflake Alert.

    CREATE OR REPLACE ALERT OBS_DB.MONITOR.ALERT_P1_P2
        WAREHOUSE = OBS_WH
        SCHEDULE = '5 MINUTE'
        IF (EXISTS (SELECT 1 FROM OBS_DB.MONITOR.V_ESCALATION_PAYLOAD WHERE PRIORITY IN ('P1', 'P2')))
        THEN CALL OBS_DB.MONITOR.SP_ESCALATE_INCIDENTS();
    ALTER ALERT OBS_DB.MONITOR.ALERT_P1_P2 RESUME;

    -- Inside SP_ESCALATE_INCIDENTS, per payload row:
    CALL SYSTEM$SEND_EMAIL('OBS_EMAIL_INT', 'ops-lead@company.com', :subject, :body);
    CALL SYSTEM$SEND_SNOWFLAKE_NOTIFICATION(
        SNOWFLAKE.NOTIFICATION.TEXT_PLAIN(SNOWFLAKE.NOTIFICATION.SANITIZE_WEBHOOK_CONTENT(:subject)),
        SNOWFLAKE.NOTIFICATION.INTEGRATION('OBS_SLACK_INT'));
    -- Then insert into ESCALATIONS and set INCIDENTS.STATUS = 'ESCALATED'.

5. Provide an acknowledgment procedure for the Operations Lead.

    CREATE OR REPLACE PROCEDURE OBS_DB.MONITOR.SP_ACK_INCIDENT(
        P_INCIDENT_ID STRING, P_PRIORITY STRING, P_ASSIGNEE STRING)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS CALLER
    AS
    BEGIN
        UPDATE OBS_DB.MONITOR.ESCALATIONS
           SET CONFIRMED_PRIORITY = :P_PRIORITY,
               ASSIGNED_TO = :P_ASSIGNEE,
               ACKNOWLEDGED_BY = CURRENT_USER(),
               ACKNOWLEDGED_AT = CURRENT_TIMESTAMP()
         WHERE INCIDENT_ID = :P_INCIDENT_ID AND ACKNOWLEDGED_AT IS NULL;
        UPDATE OBS_DB.MONITOR.INCIDENTS
           SET STATUS = 'ACKNOWLEDGED', PRIORITY = :P_PRIORITY
         WHERE INCIDENT_ID = :P_INCIDENT_ID;
        RETURN 'Incident ' || :P_INCIDENT_ID || ' acknowledged and assigned to ' || :P_ASSIGNEE;
    END;

6. Schedule a re-notification Task that resends unacknowledged P1 and P2 incidents according to the routing table, and an hourly and daily digest Task for P3 and P4.
7. Hand acknowledged incidents to the Remediation Agent by exposing a V_REMEDIATION_QUEUE view filtered to STATUS = 'ACKNOWLEDGED'.

## Coding Conventions
- Store webhook URLs and tokens only in SECRET objects; never in procedure code or payload views.
- Grant SP_ACK_INCIDENT only to an OPS_LEAD_ROLE; the agent role cannot acknowledge or assign.
- Record both AGENT_PRIORITY and CONFIRMED_PRIORITY so priority overrides feed impact weight tuning.
- Check suppression windows before every send; CI/CD and Release agents should insert windows automatically for planned deployments.
- Keep payloads under channel size limits; link to the incident record rather than embedding full evidence.
- Escalation sends are idempotent per incident and re-notify interval.