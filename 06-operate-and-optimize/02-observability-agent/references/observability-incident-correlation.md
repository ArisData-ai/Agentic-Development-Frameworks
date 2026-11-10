---
name: observability-incident-correlation
description: This skill defines how the Observability Agent groups raw signal events into incidents through deduplication, time-window clustering and upstream lineage, selects the probable root cause, scores business impact from pipeline tier and downstream consumers, and produces Cortex Agent triage summaries stored in an INCIDENTS table. Use this skill for any task involving alert deduplication, alert grouping, noise reduction, root cause identification, incident prioritization, impact scoring, Cortex COMPLETE or Cortex Agents triage, or turning many alerts into a ranked incident queue.
---
# Observability Incident Correlation

## Overview
This skill implements the Act stage. It collapses many signals into a few incidents so operators work from ranked problems, not raw alerts. Correlation runs in three passes:

    Pass            Logic                                                     Result
    -------------   -------------------------------------------------------   -----------------------------
    Deduplicate     Same pipeline + same signal type within open window       One signal family per pipeline
    Cluster         Signals on pipelines linked by UPSTREAM_IDS, 30-min       One incident per failure chain
                    window
    Prioritize      Impact score from tier, consumers, SLO breach, duration   Ranked incident queue

## When to Use
- Operators receive multiple alerts for a single upstream failure.
- Incidents need a ranked order by business impact.
- A natural-language triage summary is needed for each incident.
- Tuning grouping windows or impact weights.

## Instructions
1. Create incident tables.

    CREATE TABLE IF NOT EXISTS OBS_DB.MONITOR.INCIDENTS (
        INCIDENT_ID        STRING PRIMARY KEY,
        ROOT_PIPELINE_ID   STRING,
        AFFECTED_PIPELINES ARRAY,
        SIGNAL_TYPES       ARRAY,
        SIGNAL_COUNT       NUMBER,
        IMPACT_SCORE       NUMBER(5,2),
        PRIORITY           STRING,           -- P1 | P2 | P3 | P4
        TRIAGE_SUMMARY     STRING,
        STATUS             STRING DEFAULT 'OPEN',  -- OPEN | ESCALATED | ACKNOWLEDGED | RESOLVED
        FIRST_SIGNAL_AT    TIMESTAMP_LTZ,
        LAST_SIGNAL_AT     TIMESTAMP_LTZ,
        CREATED_AT         TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

2. Attach new signals to an existing open incident when the pipeline, or any upstream pipeline, already has one in the window.

    UPDATE OBS_DB.MONITOR.SIGNAL_EVENTS s
    SET INCIDENT_ID = i.INCIDENT_ID
    FROM OBS_DB.MONITOR.INCIDENTS i, OBS_DB.MONITOR.PIPELINE_REGISTRY r
    WHERE s.INCIDENT_ID IS NULL
      AND r.PIPELINE_ID = s.PIPELINE_ID
      AND i.STATUS IN ('OPEN', 'ESCALATED', 'ACKNOWLEDGED')
      AND s.EVENT_TIME <= DATEADD('minute', 30, i.LAST_SIGNAL_AT)
      AND (ARRAY_CONTAINS(s.PIPELINE_ID::VARIANT, i.AFFECTED_PIPELINES)
           OR ARRAYS_OVERLAP(r.UPSTREAM_IDS, i.AFFECTED_PIPELINES));

3. Open new incidents for remaining unassigned signals. Select the root as the most upstream pipeline with the earliest signal.

    CREATE OR REPLACE TEMPORARY TABLE NEW_CLUSTERS AS
    SELECT
        UUID_STRING() AS INCIDENT_ID,
        s.PIPELINE_ID AS ROOT_PIPELINE_ID,
        ARRAY_UNIQUE_AGG(s.PIPELINE_ID) AS AFFECTED_PIPELINES,
        ARRAY_UNIQUE_AGG(s.SIGNAL_TYPE) AS SIGNAL_TYPES,
        COUNT(*) AS SIGNAL_COUNT,
        MIN(s.EVENT_TIME) AS FIRST_SIGNAL_AT,
        MAX(s.EVENT_TIME) AS LAST_SIGNAL_AT
    FROM OBS_DB.MONITOR.SIGNAL_EVENTS s
    WHERE s.INCIDENT_ID IS NULL
    GROUP BY s.PIPELINE_ID;

4. Score business impact with weights stored in a config table.

    -- IMPACT_SCORE = tier_weight + consumer_weight + breach_weight + duration_weight
    SELECT
        c.INCIDENT_ID,
        DECODE(r.BUSINESS_TIER, 1, 50, 2, 25, 10)
          + LEAST(ARRAY_SIZE(c.AFFECTED_PIPELINES) * 5, 20)
          + IFF(ARRAY_CONTAINS('FAILURE'::VARIANT, c.SIGNAL_TYPES), 20, 10)
          + LEAST(DATEDIFF('minute', c.FIRST_SIGNAL_AT, CURRENT_TIMESTAMP()) / 6, 10) AS IMPACT_SCORE,
        CASE WHEN IMPACT_SCORE >= 80 THEN 'P1'
             WHEN IMPACT_SCORE >= 60 THEN 'P2'
             WHEN IMPACT_SCORE >= 40 THEN 'P3'
             ELSE 'P4' END AS PRIORITY
    FROM NEW_CLUSTERS c
    JOIN OBS_DB.MONITOR.PIPELINE_REGISTRY r ON r.PIPELINE_ID = c.ROOT_PIPELINE_ID;

5. Generate a triage summary with Cortex. Keep the prompt grounded in stored evidence only.

    UPDATE OBS_DB.MONITOR.INCIDENTS i
    SET TRIAGE_SUMMARY = SNOWFLAKE.CORTEX.COMPLETE(
        'mistral-large2',
        'You are a data platform triage assistant. Using only the evidence below, return: '
        || '1) probable root cause, 2) affected data products, 3) consumer impact, '
        || '4) recommended first check. Do not invent facts. Evidence: '
        || (SELECT TO_JSON(ARRAY_AGG(OBJECT_CONSTRUCT('pipeline', PIPELINE_ID, 'type', SIGNAL_TYPE,
                                                     'evidence', EVIDENCE, 'time', EVENT_TIME)))
            FROM OBS_DB.MONITOR.SIGNAL_EVENTS WHERE INCIDENT_ID = i.INCIDENT_ID))
    WHERE i.TRIAGE_SUMMARY IS NULL;

6. For conversational triage, expose INCIDENTS and SIGNAL_EVENTS through a semantic view and a Cortex Agent so operators can ask questions such as "which P1 incidents affect finance data products" or "what changed before this failure".
7. Package steps 2 to 5 in SP_CORRELATE_INCIDENTS and run it after SP_DETECT_SIGNALS.

## Coding Conventions
- Store grouping window, impact weights and priority cutoffs in a config table, not in code.
- An incident always has exactly one ROOT_PIPELINE_ID; downstream pipelines appear only in AFFECTED_PIPELINES.
- Never reopen a RESOLVED incident; a recurrence opens a new incident linked through a PARENT_INCIDENT_ID column if needed.
- Restrict Cortex prompts to stored evidence; select the model through a config value so it can be changed per account region and availability.
- Log every correlation run (signals in, incidents opened, incidents updated) for noise reduction reporting.