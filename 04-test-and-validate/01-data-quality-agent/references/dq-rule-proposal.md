---
name: dq-rule-proposal
description: This skill defines how the Data Quality Agent profiles monitored tables, detects columns and dimensions without rule coverage, and drafts threshold-based rules with rationale in a PROPOSED state for QA Lead approval, built on Snowpark profiling, Snowflake Horizon classification, and Cortex Agents. Use this skill for any task involving data profiling, quality rule recommendations, coverage gap analysis, auto-generating null, uniqueness, range, or accepted value rules, baselining distributions, or expanding DQ coverage on new ERP, CRM, or finance sources.
---
# DQ Rule Proposal

## Overview
Rule proposal closes coverage gaps. The agent profiles columns, compares them with existing rules, and drafts new rules grounded in observed data. Proposed rules stay inactive until the QA Lead approves them; the agent never activates its own rules.

## When to Use
- Onboarding a new table with no existing rules
- Finding columns or dimensions that lack checks
- Refreshing baselines after a source change
- Preparing a rule pack for QA Lead review

## Instructions

### 1. Profile columns
Run profiling in a Snowpark procedure over a recent sample (default last 30 days or 1,000,000 rows, whichever is smaller).

    CREATE TABLE IF NOT EXISTS DQ_DB.CONFIG.COLUMN_BASELINES (
        TABLE_FQN       STRING,
        COLUMN_NAME     STRING,
        DATA_TYPE       STRING,
        ROW_COUNT       NUMBER,
        NULL_PCT        FLOAT,
        DISTINCT_RATIO  FLOAT,
        MIN_VAL         VARIANT,
        MAX_VAL         VARIANT,
        MEAN_VAL        FLOAT,
        STDDEV_VAL      FLOAT,
        P05             FLOAT,
        P95             FLOAT,
        TOP_VALUES      VARIANT,       -- array of {value, share}
        SEMANTIC_CATEGORY STRING,      -- from Horizon classification
        PROFILED_AT     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
    );

Pull semantic categories from Horizon classification to tag identifiers, emails, and sensitive fields:

    CALL SYSTEM$CLASSIFY('RAW_DB.ERP.CUSTOMERS', {'auto_tag': false});

### 2. Detect coverage gaps
    SELECT b.TABLE_FQN, b.COLUMN_NAME, d.DIMENSION
    FROM DQ_DB.CONFIG.COLUMN_BASELINES b
    CROSS JOIN (SELECT COLUMN1 AS DIMENSION FROM VALUES
                ('COMPLETENESS'),('UNIQUENESS'),('VALIDITY'),('DISTRIBUTION')) d
    LEFT JOIN DQ_DB.CONFIG.DQ_RULES r
      ON r.TABLE_FQN = b.TABLE_FQN AND r.COLUMN_NAME = b.COLUMN_NAME
     AND r.DIMENSION = d.DIMENSION AND r.STATUS IN ('ACTIVE','PROPOSED')
    WHERE r.RULE_ID IS NULL;

### 3. Apply proposal heuristics
    COMPLETENESS : NULL_PCT = 0 over baseline           -> NULL_COUNT = 0, severity MAJOR
                   NULL_PCT < 5                         -> NULL_PERCENT <= NULL_PCT * 1.5, severity MINOR
    UNIQUENESS   : DISTINCT_RATIO = 1 or column in PK   -> DUPLICATE_COUNT = 0, severity BLOCKER
    VALIDITY     : <= 20 distinct values                -> accepted values DMF from TOP_VALUES, severity MAJOR
                   numeric amount or quantity           -> range P05 * 0.5 to P95 * 2, severity MINOR
                   SEMANTIC_CATEGORY = EMAIL            -> format regex check, severity MINOR
    DISTRIBUTION : numeric with STDDEV_VAL > 0          -> mean drift within 3 stddev, severity MINOR
    FRESHNESS    : table has load timestamp column      -> FRESHNESS <= SLA from MONITORED_TABLES, severity MAJOR

Skip proposals on columns with fewer than 100 non-null rows; mark them for manual review instead.

### 4. Insert proposals with rationale
    INSERT INTO DQ_DB.CONFIG.DQ_RULES
        (TABLE_FQN, COLUMN_NAME, DIMENSION, METRIC_FUNCTION, OPERATOR,
         THRESHOLD_LOW, THRESHOLD_HIGH, SEVERITY, STATUS, RATIONALE, PROPOSED_BY)
    VALUES
        ('RAW_DB.ERP.SALES_ORDERS', 'ORDER_ID', 'UNIQUENESS', 'SNOWFLAKE.CORE.DUPLICATE_COUNT', '=',
         0, 0, 'BLOCKER', 'PROPOSED',
         'Distinct ratio 1.0 across 30 days and column is declared primary key', 'DATA_QUALITY_AGENT');

Rationale must cite the observed statistic and the sample window.

### 5. Hand off
After inserting proposals, call SP_ESCALATE_RUN so the QA Lead receives the rule pack (see dq-qa-lead-escalation). Proposals become ACTIVE only through SP_APPROVE_RULE executed by DQ_QA_LEAD.

## Coding Conventions
- Profiling runs in Snowpark Python with pushdown aggregation; never pull full tables into memory.
- Proposals are idempotent; do not insert a duplicate PROPOSED rule for the same table, column, and dimension.
- Thresholds derive from data, never from guesses; store the baseline snapshot that justified each rule.
- Severity defaults follow the heuristic table; the QA Lead may override at approval.
- Re-profile on schema change or every 30 days and propose threshold updates as new PROPOSED rules that supersede old ones.