# Data Assessment Agent

The Data Assessment Agent evaluates the readiness of candidate data sources so teams know what is usable before design starts. It profiles each source, flags sensitive data, scores migration readiness and risk, and sends the findings to the Lead Validator for a decision.

**Segment:** 1, Discover & Assess
**Human owner:** Lead Validator

## How It Works

1. **Sense:** Detects newly discovered sources entering the assessment queue.
2. **Act:** Profiles completeness, consistency, volume, and sensitivity, then scores each source for migration readiness and risk.
3. **Hand off:** Delivers scored findings to the Lead Validator, who accepts, downgrades, or sends sources back for remediation.

## Skill Files

| File | Purpose |
| --- | --- |
| `SKILL.md` | Primary skill covering the end-to-end workflow and operating rules |
| `data-source-profiling.md` | Snowpark profiling for completeness, consistency, and volume |
| `data-sensitivity-classification.md` | Snowflake Horizon classification, tagging, and sensitivity findings |
| `migration-readiness-scoring.md` | Readiness scores, risk levels, and readiness bands |
| `lead-validator-handoff.md` | Review packages, validator decisions, and Cortex Analyst reporting |

Start with `data-assessment-agent.md`, then use the sibling skill for the task at hand.

## Folder Structure

    data-assessment-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
       └── data-source-profiling.md
       └── data-sensitivity-classification.md
       └── migration-readiness-scoring.md
       └── lead-validator-handoff.md

## Snowflake Components

| Component | Role |
| --- | --- |
| Snowpark | Runs profiling logic close to the data |
| Snowflake Horizon | Classifies sensitive data and tags it for governance |
| Cortex Analyst | Surfaces readiness scores and trends across sources |
| Streams & Tasks | Detect queued sources and start assessment runs |

## Prerequisites

- A dedicated assessment role, warehouse, and `ASSESSMENT` schema
- Read-only access to candidate source objects
- Privileges for Snowflake Horizon classification, including the `SNOWFLAKE.CLASSIFICATION_ADMIN` database role
- Required columns, expected row count ranges, and consistency rules for each source
- A named Lead Validator for each assessment

## Setup

1. Create the assessment schema, role, and warehouse.
2. Create the queue, run, stream, and task objects from `data-assessment-agent.md`.
3. Create the remaining objects listed under Before You Deploy.
4. Configure the source inputs, scoring weights, thresholds, and risk rules.
5. Test the workflow with a non-production source.
6. Resume the task:

        ALTER TASK ASSESSMENT.RUN_ASSESSMENT RESUME;

7. Add a source to the queue, then review the results with the Lead Validator.

## Outputs

- Column-level profiling metrics and consistency rule results
- Sensitivity findings and system classification tags
- Readiness scores, risk levels, and readiness bands
- Validator decisions with a rationale and decision maker
- An approved source list for Segment 2 design work

## Before You Deploy

### Confirm the defaults

The source reference does not define the items below. Treat them as starting defaults, and confirm or replace them for your deployment:

- Table names
- Stream and task setup
- Scoring weights
- Readiness thresholds
- Risk rules
- Example source systems

Agree on the scoring weights, readiness thresholds, and risk rules with the Lead Validator before relying on the results.

### Create the remaining objects

The skills do not provide complete definitions for these objects:

- `ASSESSMENT.RUN_ASSESSMENT_BATCH`
- `ASSESSMENT.DIMENSION_SCORES`
- `ASSESSMENT.PROFILE_RESULTS`
- `ASSESSMENT.RULE_RESULTS`
- `ASSESSMENT.SENSITIVITY_FINDINGS`
- `ASSESSMENT.ASSESSMENT_SCORES`

### Validate the implementation

The SQL and Snowpark examples have not been run in a Snowflake account. Test them in a non-production environment, and confirm the current syntax, edition requirements, and privileges for Snowflake Horizon classification and semantic views.

## Operating Rules

- The agent recommends, and the Lead Validator decides.
- Sources do not move to design until a decision is recorded.
- A failed or skipped check never counts as a pass.
- Assessment does not change source data or fix quality issues.
- Sample values from sensitive columns are not stored.
- Each reassessment uses a new `RUN_ID` and keeps earlier results.
- The Governance Agent owns masking and access policies.