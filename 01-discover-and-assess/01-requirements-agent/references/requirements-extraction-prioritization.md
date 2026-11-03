---
name: requirements-extraction-prioritization
description: This skill defines how the Requirements Agent extracts functional and data requirements from indexed intake content, flags conflicts and gaps, and ranks each requirement against stated business objectives through Cortex Agents, Cortex Search, and Snowflake AI functions. Use this skill for any task involving requirement extraction, deduplication, conflict detection, gap analysis, acceptance criteria drafting, objective traceability, prioritization, or draft requirements baseline generation.
---

# Requirements Extraction and Prioritization

## Overview

This skill covers the Act phase. It converts indexed content into structured requirements, exposes contradictions and missing information, and produces a ranked draft for human review.

## When to Use

- Extract requirements from new intake material.
- Separate functional requirements from data requirements.
- Identify duplicates, conflicts, ambiguity, or missing details.
- Rank requirements against program objectives.

## Instructions

1. Retrieve relevant chunks through `INTAKE_SEARCH`, filtered by `PROGRAM_ID`.
2. Request structured output from `AI_COMPLETE` with an approved model and this contract:

        {
          "requirements": [
            {
              "type": "FUNCTIONAL | DATA",
              "statement": "The system must ...",
              "acceptance_criteria": "string or null",
              "objective_id": "string or null",
              "source_chunk_ids": ["string"],
              "confidence": 0.0
            }
          ]
        }

3. Reject any item that has no valid `source_chunk_ids`.
4. Write each requirement as one testable statement. Preserve business terms from the source.
5. Store findings in `REQUIREMENT_FINDING` with these types:

   - `CONTRADICTION`: Requirements or sources disagree.
   - `DUPLICATE`: Statements express the same need.
   - `AMBIGUOUS`: The statement cannot be tested as written.
   - `MISSING_ACCEPTANCE_CRITERIA`: No measurable outcome exists.
   - `NO_OBJECTIVE_LINK`: No stated objective supports the requirement.

6. Link duplicates to a proposed primary requirement, but retain every source reference.
7. Preserve both sides of a contradiction for the Lead Validator.
8. Rank requirements using configurable weights.

        UPDATE REQUIREMENT r
        SET PRIORITY_SCORE = s.SCORE,
            PRIORITY_RANK = s.SCORE_RANK
        FROM (
            SELECT
                req.REQUIREMENT_ID,
                (
                    0.50 * COALESCE(o.OBJECTIVE_WEIGHT, 0)
                    + 0.30 * req.MANDATE_FLAG
                    + 0.20 * req.DEPENDENCY_FACTOR
                ) * 100 AS SCORE,
                RANK() OVER (
                    PARTITION BY req.PROGRAM_ID
                    ORDER BY SCORE DESC
                ) AS SCORE_RANK
            FROM REQUIREMENT_SCORING_INPUT req
            LEFT JOIN BUSINESS_OBJECTIVE o
                ON req.OBJECTIVE_ID = o.OBJECTIVE_ID
        ) s
        WHERE r.REQUIREMENT_ID = s.REQUIREMENT_ID;

9. Store a short ranking rationale with each requirement.
10. Set completed requirements to `DRAFT` and leave final decisions to the Lead Validator.

## Coding Conventions

- Validate model output against the JSON contract before writing to tables.
- Do not invent acceptance criteria, owners, or objectives.
- Treat confidence as a review signal, never as approval.
- Keep scoring weights in a configuration table for each program.
- Retain the model name, prompt version, and extraction timestamp.