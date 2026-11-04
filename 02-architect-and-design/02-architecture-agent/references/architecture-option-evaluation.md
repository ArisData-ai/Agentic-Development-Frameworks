---
name: architecture-option-evaluation
description: This skill defines a weighted evaluation method that compares Snowflake ingestion, transformation, and consumption options against workload, scale, latency, cost, and operational constraints, and outputs scored options with documented trade-offs. Use this skill for any task involving comparing Openflow, Snowpipe, Snowpipe Streaming, or COPY INTO, choosing Dynamic Tables, Streams and Tasks, or Snowpark, scoring architecture options, building a trade-off matrix, or justifying an integration pattern.
---

# Architecture Option Evaluation

## Overview

This skill turns architecture choices into a consistent, evidence-based comparison. It helps the Lead Architect see why an option ranks highly and what the team gives up by choosing it.

## When to Use

Use this skill when more than one pattern could meet a requirement, or when a reviewer asks for the reasoning behind a recommendation. Skip the weighted scoring only when a hard constraint leaves one viable option, and record that constraint.

## Instructions

1. Apply hard constraints first. Reject any option that fails a mandatory security, residency, latency, or source-connectivity requirement.
2. Evaluate each architecture layer separately, then confirm that the selected options work together.
3. Compare options that fit the workload:

| Layer | Candidate options | Favor when |
| --- | --- | --- |
| Ingestion | Openflow | Managed connectivity is needed across source systems |
| Ingestion | Snowpipe | Files arrive continuously in cloud storage |
| Ingestion | Snowpipe Streaming | Rows require low-latency streaming ingestion |
| Ingestion | COPY INTO | Loads are batch-based or one-time |
| Transformation | Dynamic Tables | Declarative pipelines can meet a target lag |
| Transformation | Streams & Tasks | Change handling needs explicit procedural control |
| Transformation | Snowpark | Logic is complex or better suited to Python, Java, or Scala |
| Consumption | Secure views | Consumers need governed SQL or BI access |
| Consumption | Semantic views with Cortex Analyst | Business users need natural-language analytics |
| Consumption | Secure Data Sharing | Other Snowflake accounts need live data access |

4. Use these default criteria and weights. Change them only when the request gives a reason, and record that reason.

| Criterion | Default weight |
| --- | --- |
| Requirement and latency fit | 0.30 |
| Scalability and reliability | 0.20 |
| Security and compliance fit | 0.20 |
| Total cost | 0.15 |
| Operational complexity | 0.10 |
| Team skills and adoption effort | 0.05 |

5. Score each criterion from 1 to 5, with 5 as the best outcome. Cite evidence for each score.
6. Mark an estimate as `ASSUMED` when evidence is unavailable. Do not present estimated credits, throughput, or latency as measured values.
7. Treat scores within 0.25 of each other as a tie. Present the tied options to the Lead Architect with the deciding trade-off.
8. Record what each option gains, what it gives up, its main risks, and the conditions that would change the ranking.

Store and rank scores with this pattern:

    CREATE TABLE IF NOT EXISTS ARCH_DB.DECISIONS.ARCH_OPTION_SCORE (
        REQUEST_ID    STRING NOT NULL,
        OPTION_ID     STRING NOT NULL,
        LAYER         STRING NOT NULL,
        PATTERN_NAME  STRING NOT NULL,
        CRITERION     STRING NOT NULL,
        WEIGHT        NUMBER(4,2) NOT NULL,
        SCORE         NUMBER(1,0) NOT NULL,
        EVIDENCE_TYPE STRING NOT NULL,   -- MEASURED | DOCUMENTED | ASSUMED
        EVIDENCE_REF  STRING,
        RATIONALE     STRING NOT NULL
    );

    SELECT
        REQUEST_ID,
        LAYER,
        OPTION_ID,
        PATTERN_NAME,
        ROUND(SUM(WEIGHT * SCORE), 2) AS WEIGHTED_SCORE,
        COUNT_IF(EVIDENCE_TYPE = 'ASSUMED') AS ASSUMED_CRITERIA
    FROM ARCH_DB.DECISIONS.ARCH_OPTION_SCORE
    WHERE REQUEST_ID = :request_id
    GROUP BY REQUEST_ID, LAYER, OPTION_ID, PATTERN_NAME
    ORDER BY LAYER, WEIGHTED_SCORE DESC;

## Coding Conventions

- Confirm that weights total 1.00 for each layer and option.
- Use the same criteria for every option in a layer.
- Store one row per option and criterion.
- Use stable values such as `INGESTION`, `TRANSFORMATION`, and `CONSUMPTION`.
- Do not change historic scores; add a new evaluation version instead.