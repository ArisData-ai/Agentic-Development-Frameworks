---
name: remediation-recurrence-tracking
description: This skill defines recurrence tracking for the Remediation Agent, which aggregates incident and proposal outcomes into MTTR, auto-resolution rate, and recurring root cause metrics and flags repeat offenders for permanent fixes through views, Cortex AI functions, and scheduled reports. Use this skill for any task involving MTTR reporting, repeat incident analysis, recurring failure clustering, on-call fatigue metrics, permanent fix backlogs, Jira or ServiceNow problem tickets, remediation dashboards, or measuring agent effectiveness for operations managers.
---
# Remediation Recurrence Tracking

## Overview
Tracking closes the loop for operations managers. It measures how fast incidents recover, how often the agent resolves them safely, and which causes keep returning so teams fund permanent fixes instead of repeated retries.

## When to Use
- Building MTTR and auto-resolution reporting
- Identifying recurring causes and repeat-offender objects
- Generating problem tickets or permanent fix backlogs

## Instructions
1. Create a metrics view joining incidents and proposals.

        CREATE OR REPLACE VIEW OPS_DB.REMEDIATION.V_REMEDIATION_METRICS AS
        SELECT
            i.incident_id, i.object_name, i.source_type, i.severity, i.error_code,
            p.root_cause, p.action_type, p.risk_tier, p.status, p.outcome,
            i.detected_at, p.decided_at, p.executed_at,
            DATEDIFF('minute', i.detected_at, p.executed_at) AS minutes_to_recover,
            IFF(p.risk_tier = 'SAFE' AND p.outcome = 'RESOLVED', 1, 0) AS auto_resolved
        FROM OPS_DB.REMEDIATION.INCIDENTS i
        LEFT JOIN OPS_DB.REMEDIATION.REMEDIATION_PROPOSALS p
               ON p.incident_id = i.incident_id;

2. Report weekly MTTR and auto-resolution rate by severity.

        SELECT DATE_TRUNC('week', detected_at) AS week, severity,
               AVG(minutes_to_recover)  AS mttr_minutes,
               AVG(auto_resolved)       AS auto_resolution_rate,
               COUNT(*)                 AS incidents
        FROM OPS_DB.REMEDIATION.V_REMEDIATION_METRICS
        GROUP BY 1, 2
        ORDER BY 1 DESC, 2;

3. Flag recurring causes for permanent fixes.

        SELECT object_name, error_code, root_cause, COUNT(*) AS occurrences_30d
        FROM OPS_DB.REMEDIATION.V_REMEDIATION_METRICS
        WHERE detected_at > DATEADD('day', -30, CURRENT_TIMESTAMP())
        GROUP BY 1, 2, 3
        HAVING COUNT(*) >= 3
        ORDER BY occurrences_30d DESC;

4. Summarize recurring clusters with a Cortex AI function for the problem ticket body.

        SELECT object_name,
               SNOWFLAKE.CORTEX.COMPLETE('claude-4-sonnet',
                   'Summarize the recurring failure pattern and recommend a permanent fix: '
                   || LISTAGG(error_message, ' | ')) AS permanent_fix_summary
        FROM OPS_DB.REMEDIATION.INCIDENTS
        WHERE detected_at > DATEADD('day', -30, CURRENT_TIMESTAMP())
        GROUP BY object_name
        HAVING COUNT(*) >= 3;

5. Schedule a weekly Task that writes flagged items to a PERMANENT_FIX_BACKLOG table and optionally opens Jira or ServiceNow tickets through an external access integration.
6. Publish the view to a Streamlit or Snowsight dashboard for service owners and operations managers.

## Coding Conventions
- Compute metrics from views, never from copies of incident data.
- Treat three or more occurrences in 30 days as the default recurrence threshold; make it configurable.
- Exclude test and non-production objects from executive metrics with an environment filter.
- Keep ticket creation idempotent by keying on object_name plus error_code.