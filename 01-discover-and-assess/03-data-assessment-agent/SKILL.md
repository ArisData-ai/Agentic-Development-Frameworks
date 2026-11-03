---
name: data-assessment-agent
description: This skill defines the end-to-end Data Assessment Agent workflow that takes candidate sources from an assessment queue, coordinates Snowpark profiling, Snowflake Horizon sensitivity classification, and readiness scoring, and produces scored findings for Lead Validator review. Use this skill for any task involving building, running, or troubleshooting a data readiness assessment, including source triage, migration readiness checks, pre-design data evaluation, assessment queue setup, and legacy assessments of Oracle, SQL Server, Teradata, SAP, or Salesforce sources.
---

# Data Assessment Agent

## Overview

The Data Assessment Agent evaluates candidate data sources so teams know what is usable before design starts. It belongs to Segment 1: Discover & Assess. It receives sources from the Discovery Agent, assesses them, and delivers scored findings to the Lead Validator, who owns the final decision.

The workflow follows three stages:

- Sense: Detect newly discovered sources entering the assessment queue.
- Act: Profile completeness, consistency, volume, and sensitivity, then score migration readiness and risk.
- Hand off: Deliver findings to the Lead Validator, who accepts, downgrades, or returns each source for remediation.

### Implementation Note

Review the following before you build or deploy this agent:

- Reference content: The source reference does not define the table names, stream and task setup, scoring weights, readiness thresholds, risk rules, or example source systems used in these skills. Treat them as starting defaults, and confirm or replace them for your deployment.
- Calibration: Agree on the scoring weights, readiness thresholds, and risk rules with the Lead Validator before relying on the results.
- Objects to create: Build ASSESSMENT.RUN_ASSESSMENT_BATCH, ASSESSMENT.DIMENSION_SCORES, ASSESSMENT.PROFILE_RESULTS, ASSESSMENT.RULE_RESULTS, ASSESSMENT.SENSITIVITY_FINDINGS, and ASSESSMENT.ASSESSMENT_SCORES because these skills do not provide their complete definitions.
- Required inputs: Provide required columns, expected row count ranges, and consistency rules for each source, because the scoring logic depends on them.
- Task activation: Run ALTER TASK ASSESSMENT.RUN_ASSESSMENT RESUME after setup because new tasks start in a suspended state.
- Validation: The SQL and Snowpark examples have not been run in a Snowflake account. Test them in a non-production environment, and confirm the current syntax, edition requirements, and privileges for Snowflake Horizon classification and semantic views.

## When to Use

- Set up or modify the end-to-end assessment workflow.
- Assess new sources before data modeling or architecture work begins.
- Investigate a failed, stalled, or incomplete assessment run.
- Determine which sibling skill applies to a specific assessment task.

## Instructions

1. Create the queue and the workflow objects.

        CREATE TABLE IF NOT EXISTS ASSESSMENT.ASSESSMENT_QUEUE (
            SOURCE_ID      STRING NOT NULL,
            SOURCE_SYSTEM  STRING NOT NULL,
            OBJECT_FQN     STRING NOT NULL,
            DATA_OWNER     STRING,
            DISCOVERED_AT  TIMESTAMP_NTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS ASSESSMENT.ASSESSMENT_RUNS (
            RUN_ID         STRING NOT NULL,
            SOURCE_ID      STRING NOT NULL,
            STATUS         STRING NOT NULL,
            FAILURE_REASON STRING,
            STARTED_AT     TIMESTAMP_NTZ,
            COMPLETED_AT   TIMESTAMP_NTZ
        );

        CREATE OR REPLACE STREAM ASSESSMENT.QUEUE_STREAM
            ON TABLE ASSESSMENT.ASSESSMENT_QUEUE
            APPEND_ONLY = TRUE;

        CREATE OR REPLACE TASK ASSESSMENT.RUN_ASSESSMENT
            WAREHOUSE = ASSESSMENT_WH
            SCHEDULE = '15 MINUTES'
            WHEN SYSTEM$STREAM_HAS_DATA('ASSESSMENT.QUEUE_STREAM')
        AS
            CALL ASSESSMENT.RUN_ASSESSMENT_BATCH();

2. Consume the stream in a DML statement inside the batch procedure so that processed rows advance the stream offset.
3. Assign a unique RUN_ID to each source and record the run as RUNNING.
4. Profile the source with the data-source-profiling skill.
5. Classify the source with the data-sensitivity-classification skill.
6. Calculate scores with the migration-readiness-scoring skill.
7. Publish findings with the lead-validator-handoff skill.
8. Mark the run as AWAITING_REVIEW, or as FAILED with a recorded reason.

Apply these operating rules:

- Assess every queued source independently so that one failure does not stop the batch.
- Create a new RUN_ID for every reassessment and preserve earlier results.
- Do not mark a source as ready when any assessment stage failed or was skipped.
- Do not pass a source to design until the Lead Validator records a decision.
- Do not change source data or fix quality issues during assessment.

## Coding Conventions

- Use uppercase Snowflake identifiers and fully qualified object names.
- Key all result tables by RUN_ID and SOURCE_ID.
- Store timestamps in TIMESTAMP_NTZ using UTC.
- Use a dedicated role and warehouse for assessment workloads.
- Grant read-only access to source objects.
- Log the stage, status, and failure reason for each run.

## References

- `references/data-source-profiling.md`: Snowpark profiling for completeness, consistency, and volume.
- `references/data-sensitivity-classification.md`: Snowflake Horizon classification, tagging, and sensitivity findings.
- `references/migration-readiness-scoring.md`: Readiness scores, risk levels, and readiness bands.
- `references/lead-validator-handoff.md`: Findings packages, validator decisions, and Cortex Analyst reporting.