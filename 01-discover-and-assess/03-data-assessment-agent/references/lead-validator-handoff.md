---
name: lead-validator-handoff
description: This skill defines how scored assessment findings reach the Lead Validator, how accept, downgrade, and remediation decisions are recorded, and how Cortex Analyst surfaces readiness scores and trends across sources. Use this skill for any task involving assessment review packages, human approval workflows, validator decisions, remediation requests, source sign-off, readiness dashboards, semantic views, or natural language questions about readiness and risk.
---

# Lead Validator Handoff

## Overview

This skill keeps human accountability at the end of each assessment. The agent recommends, and the Lead Validator decides. Each decision is recorded with its rationale, and approved results become available for downstream design work and Cortex Analyst reporting.

## When to Use

- Build or update the Lead Validator review queue.
- Record accept, downgrade, or remediation decisions.
- Control which sources can move into design.
- Expose readiness scores and trends through Cortex Analyst.

## Instructions

1. Include these items in each review package:
    - Source identity, owner, RUN_ID, and SCORE_VERSION.
    - Dimension scores, readiness score, band, and risk level.
    - Main score drivers and failed or not evaluated checks.
    - Sensitive fields, with their categories and confidence levels.
    - A recommended action, labeled as an agent recommendation.
2. Record every decision in an append-only table.

        CREATE TABLE IF NOT EXISTS ASSESSMENT.VALIDATOR_DECISIONS (
            RUN_ID      STRING NOT NULL,
            SOURCE_ID   STRING NOT NULL,
            DECISION    STRING NOT NULL,
            FINAL_BAND  STRING NOT NULL,
            RATIONALE   STRING NOT NULL,
            DECIDED_BY  STRING NOT NULL,
            DECIDED_AT  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        );

3. Apply the decision to the source:
    - ACCEPT: Make the source available to Segment 2 design work.
    - DOWNGRADE: Record the lower final band and any conditions for downstream use.
    - RETURN_FOR_REMEDIATION: Hold the source, record the required fixes, and requeue it under a new RUN_ID after remediation.
4. Expose only reviewed sources to downstream consumers.

        CREATE OR REPLACE VIEW ASSESSMENT.APPROVED_SOURCES AS
        SELECT
            s.SOURCE_ID, s.SOURCE_SYSTEM, s.READINESS_SCORE, s.RISK_LEVEL,
            d.DECISION, d.FINAL_BAND, d.RATIONALE, d.DECIDED_BY, d.DECIDED_AT
        FROM ASSESSMENT.ASSESSMENT_SCORES s
        JOIN ASSESSMENT.VALIDATOR_DECISIONS d
            ON s.RUN_ID = d.RUN_ID
            AND s.SOURCE_ID = d.SOURCE_ID
        WHERE d.DECISION IN ('ACCEPT', 'DOWNGRADE')
        QUALIFY ROW_NUMBER() OVER (
            PARTITION BY s.SOURCE_ID ORDER BY d.DECIDED_AT DESC
        ) = 1;

5. Create a semantic view so Cortex Analyst can answer readiness questions.

        CREATE OR REPLACE SEMANTIC VIEW ASSESSMENT.READINESS_SV
            TABLES (
                SCORES AS ASSESSMENT.ASSESSMENT_SCORES
                    PRIMARY KEY (RUN_ID, SOURCE_ID)
            )
            FACTS (
                SCORES.READINESS_VALUE AS READINESS_SCORE
            )
            DIMENSIONS (
                SCORES.SOURCE_NAME AS SOURCE_SYSTEM,
                SCORES.RISK AS RISK_LEVEL,
                SCORES.BAND AS READINESS_BAND
            )
            METRICS (
                SCORES.AVERAGE_READINESS AS AVG(SCORES.READINESS_VALUE),
                SCORES.SOURCE_COUNT AS COUNT(SCORES.SOURCE_ID)
            )
            COMMENT = 'Readiness scores and risk levels for assessed sources';

6. Test Cortex Analyst with typical questions, such as which sources are not ready, how readiness differs by source system, and how scores changed across runs.

## Coding Conventions

- Require a rationale and named decision maker for every decision.
- Never overwrite an earlier decision; add a new record instead.
- Never let the agent record a decision on behalf of the Lead Validator.
- Filter by the latest RUN_ID when reporting current readiness.
- Limit review data to the access level of each reviewer.