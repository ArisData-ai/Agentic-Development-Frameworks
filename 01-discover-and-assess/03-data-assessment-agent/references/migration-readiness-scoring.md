---
name: migration-readiness-scoring
description: This skill defines how profiling metrics and sensitivity findings become versioned readiness scores, risk levels, and readiness bands for each candidate source in Snowflake. Use this skill for any task involving migration readiness scores, source risk ratings, scoring weights, readiness thresholds, source ranking, migration prioritization, or calculating READY, CONDITIONAL, and NOT_READY outcomes.
---

# Migration Readiness Scoring

## Overview

This skill converts assessment results into an objective, repeatable view of each source. Readiness reflects data condition, while risk reflects sensitivity. The two measures remain separate so that a high-quality sensitive source still receives appropriate review.

## When to Use

- Calculate readiness scores and risk levels.
- Change scoring weights or readiness thresholds.
- Rank or compare candidate sources.
- Explain why a source received a specific result.

## Instructions

1. Calculate each dimension on a scale of 0 to 100:
    - Completeness: 100 minus the average null percentage across required columns.
    - Consistency: Percentage of checked records that passed consistency rules.
    - Volume: 100 within the expected range, 50 outside it, and 0 for an empty source.
2. Apply the default risk rules:
    - HIGH: Any IDENTIFIER, SENSITIVE, or UNKNOWN finding.
    - MEDIUM: Any QUASI_IDENTIFIER finding.
    - LOW: No sensitive finding.
3. Calculate the score and band. Treat these default weights and thresholds as configurable, and calibrate them with the Lead Validator for each engagement.

        INSERT INTO ASSESSMENT.ASSESSMENT_SCORES (
            RUN_ID, SOURCE_ID, SOURCE_SYSTEM,
            COMPLETENESS_SCORE, CONSISTENCY_SCORE, VOLUME_SCORE,
            READINESS_SCORE, RISK_LEVEL, READINESS_BAND,
            SCORE_VERSION, SCORED_AT
        )
        WITH SCORED AS (
            SELECT
                RUN_ID, SOURCE_ID, SOURCE_SYSTEM,
                COMPLETENESS_SCORE, CONSISTENCY_SCORE, VOLUME_SCORE, RISK_LEVEL,
                ROUND(
                    0.45 * COMPLETENESS_SCORE
                    + 0.45 * CONSISTENCY_SCORE
                    + 0.10 * VOLUME_SCORE, 1
                ) AS READINESS_SCORE
            FROM ASSESSMENT.DIMENSION_SCORES
            WHERE RUN_ID = :RUN_ID
        )
        SELECT
            RUN_ID, SOURCE_ID, SOURCE_SYSTEM,
            COMPLETENESS_SCORE, CONSISTENCY_SCORE, VOLUME_SCORE,
            READINESS_SCORE, RISK_LEVEL,
            CASE
                WHEN READINESS_SCORE < 60 THEN 'NOT_READY'
                WHEN READINESS_SCORE < 85 OR RISK_LEVEL = 'HIGH' THEN 'CONDITIONAL'
                ELSE 'READY'
            END AS READINESS_BAND,
            'v1' AS SCORE_VERSION,
            CURRENT_TIMESTAMP() AS SCORED_AT
        FROM SCORED;

4. Assign NOT_READY when any required dimension is missing, and record the reason.
5. Add the main drivers behind each score, such as high null rates, failed key checks, or sensitive columns.

## Coding Conventions

- Store weights, thresholds, and SCORE_VERSION in configuration rather than scattered logic.
- Increase SCORE_VERSION whenever the scoring method changes.
- Preserve dimension scores so that every result can be explained.
- Keep calculations deterministic for the same inputs and version.
- Do not compare scores across versions without noting the change.