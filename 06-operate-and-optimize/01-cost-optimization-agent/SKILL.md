---
name: cost-optimization-agent
description: This skill defines the end-to-end Cost Optimization Agent for Snowflake, which senses credit consumption signals from ACCOUNT_USAGE views, models the savings impact of candidate changes with Snowpark, and delivers ranked, approval-gated recommendations through Cortex Agents and Cortex Analyst. Use this skill for any task involving building, configuring, or orchestrating a FinOps agent, such as designing the sense, act, and hand-off workflow, provisioning the cost schema and roles, scheduling cost scans, reducing Snowflake credit spend, or routing savings actions to an Operations Lead for approval.
---

# Cost Optimization Agent

## Overview
The Cost Optimization Agent brings continuous FinOps discipline to Snowflake credit usage. It runs a three-stage loop:

- Sense: detects usage spikes, idle warehouses, and inefficient queries from SNOWFLAKE.ACCOUNT_USAGE metadata.
- Act: quantifies waste, models the impact of changes such as warehouse resizing or auto-suspend tuning, and drafts optimization recommendations.
- Hand off: presents ranked actions to the Operations Lead, who approves, adjusts, or declines each change. The agent never applies a change without recorded approval.

Primary beneficiaries are FinOps and finance teams (spend driver visibility), platform owners (cost control without throttling workloads), and budget holders (early overrun warnings). The ideal customer is a fast-growing Snowflake account with rising, unpredictable credit bills and limited visibility into who drives them.

## When to Use
- Standing up a new FinOps agent in a customer or internal Snowflake account.
- Defining the shared schema, roles, and task schedule that all cost sub-skills depend on.
- Orchestrating the sense, act, and hand-off sequence across sibling skills.
- Reviewing agent guardrails, approval policy, or the audit trail.

## Instructions
1. Provision the foundation: a dedicated database, schema, warehouse, and least-privilege role.

        CREATE DATABASE IF NOT EXISTS COST_OPS_DB;
        CREATE SCHEMA IF NOT EXISTS COST_OPS_DB.FINOPS;
        CREATE WAREHOUSE IF NOT EXISTS FINOPS_WH
            WAREHOUSE_SIZE = XSMALL AUTO_SUSPEND = 60 AUTO_RESUME = TRUE
            INITIALLY_SUSPENDED = TRUE;

        CREATE ROLE IF NOT EXISTS FINOPS_AGENT_ROLE;
        CREATE ROLE IF NOT EXISTS FINOPS_APPROVER_ROLE;
        GRANT DATABASE ROLE SNOWFLAKE.USAGE_VIEWER      TO ROLE FINOPS_AGENT_ROLE;
        GRANT DATABASE ROLE SNOWFLAKE.GOVERNANCE_VIEWER TO ROLE FINOPS_AGENT_ROLE;
        GRANT USAGE ON WAREHOUSE FINOPS_WH TO ROLE FINOPS_AGENT_ROLE;
        GRANT USAGE ON DATABASE COST_OPS_DB TO ROLE FINOPS_AGENT_ROLE;
        GRANT ALL ON SCHEMA COST_OPS_DB.FINOPS TO ROLE FINOPS_AGENT_ROLE;
        GRANT ROLE FINOPS_AGENT_ROLE TO ROLE FINOPS_APPROVER_ROLE;

2. Create the three core tables shared by every sub-skill.

        CREATE TABLE IF NOT EXISTS COST_OPS_DB.FINOPS.COST_SIGNALS (
            SIGNAL_ID        STRING DEFAULT UUID_STRING(),
            SIGNAL_TYPE      STRING,      -- SPIKE | IDLE_WAREHOUSE | INEFFICIENT_QUERY | BUDGET_RISK
            ENTITY_TYPE      STRING,      -- WAREHOUSE | QUERY_PATTERN | USER | ROLE
            ENTITY_NAME      STRING,
            OBSERVED_CREDITS NUMBER(18,4),
            BASELINE_CREDITS NUMBER(18,4),
            DETAILS          VARIANT,
            DETECTED_AT      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS (
            REC_ID                   STRING DEFAULT UUID_STRING(),
            SIGNAL_ID                STRING,
            ACTION_TYPE              STRING,   -- RESIZE | AUTO_SUSPEND | MULTI_CLUSTER | QUERY_REWRITE | RESOURCE_MONITOR
            TARGET_OBJECT            STRING,
            CURRENT_CONFIG           VARIANT,
            PROPOSED_CONFIG          VARIANT,
            PROPOSED_SQL             STRING,
            EST_MONTHLY_SAVINGS_CR   NUMBER(18,4),
            PERFORMANCE_RISK         STRING,   -- LOW | MEDIUM | HIGH
            CONFIDENCE               NUMBER(5,2),
            PRIORITY_SCORE           NUMBER(10,4),
            STATUS                   STRING DEFAULT 'PENDING', -- PENDING | APPROVED | ADJUSTED | DECLINED | APPLIED | ROLLED_BACK
            CREATED_AT               TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

        CREATE TABLE IF NOT EXISTS COST_OPS_DB.FINOPS.COST_ACTION_LOG (
            LOG_ID        STRING DEFAULT UUID_STRING(),
            REC_ID        STRING,
            DECISION      STRING,
            DECIDED_BY    STRING DEFAULT CURRENT_USER(),
            FINAL_SQL     STRING,
            NOTES         STRING,
            DECIDED_AT    TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
        );

3. Run the workflow in this order, each step owned by a sibling skill:
    1. Sense spikes and budget risk (cost-usage-anomaly-detection).
    2. Sense idle and oversized warehouses (cost-warehouse-efficiency-analysis).
    3. Sense inefficient query patterns (cost-query-efficiency-analysis).
    4. Model savings and performance risk for each signal (cost-savings-impact-modeling).
    5. Rank, present, and record decisions; apply only approved changes (cost-recommendation-handoff).
    6. Expose trends and the conversational interface (cost-analyst-and-agent-interface).

4. Schedule the sense and act stages as a task graph on FINOPS_WH.

        CREATE OR REPLACE TASK COST_OPS_DB.FINOPS.T_COST_SCAN_ROOT
            WAREHOUSE = FINOPS_WH
            SCHEDULE = 'USING CRON 0 6 * * * UTC'
        AS CALL COST_OPS_DB.FINOPS.SP_DETECT_USAGE_SPIKES();

        CREATE OR REPLACE TASK COST_OPS_DB.FINOPS.T_WAREHOUSE_SCAN
            WAREHOUSE = FINOPS_WH AFTER COST_OPS_DB.FINOPS.T_COST_SCAN_ROOT
        AS CALL COST_OPS_DB.FINOPS.SP_DETECT_IDLE_WAREHOUSES();

        CREATE OR REPLACE TASK COST_OPS_DB.FINOPS.T_QUERY_SCAN
            WAREHOUSE = FINOPS_WH AFTER COST_OPS_DB.FINOPS.T_COST_SCAN_ROOT
        AS CALL COST_OPS_DB.FINOPS.SP_DETECT_INEFFICIENT_QUERIES();

        CREATE OR REPLACE TASK COST_OPS_DB.FINOPS.T_MODEL_AND_RANK
            WAREHOUSE = FINOPS_WH
            AFTER COST_OPS_DB.FINOPS.T_WAREHOUSE_SCAN, COST_OPS_DB.FINOPS.T_QUERY_SCAN
        AS CALL COST_OPS_DB.FINOPS.SP_MODEL_AND_RANK_RECOMMENDATIONS();

        ALTER TASK COST_OPS_DB.FINOPS.T_MODEL_AND_RANK RESUME;
        ALTER TASK COST_OPS_DB.FINOPS.T_QUERY_SCAN     RESUME;
        ALTER TASK COST_OPS_DB.FINOPS.T_WAREHOUSE_SCAN RESUME;
        ALTER TASK COST_OPS_DB.FINOPS.T_COST_SCAN_ROOT RESUME;

5. Enforce guardrails:
    - FINOPS_AGENT_ROLE holds read access to usage metadata and write access only to COST_OPS_DB.FINOPS; it holds no MODIFY on production warehouses.
    - Only the approval procedure, executed by FINOPS_APPROVER_ROLE, runs ALTER WAREHOUSE statements.
    - Every decision writes to COST_ACTION_LOG with the approver identity and final SQL.
    - Account for ACCOUNT_USAGE latency (up to roughly 3 hours); never treat the most recent 3 hours as complete.

## Coding Conventions
- Fully qualify every object as DATABASE.SCHEMA.OBJECT.
- Use UPPER_SNAKE_CASE for objects; prefix procedures with SP_, tasks with T_, views with V_.
- Express credits as NUMBER(18,4); convert to currency only in presentation layers using a configurable CREDIT_PRICE parameter.
- Store recommendation context in VARIANT columns, never in free-text notes.
- Keep detection logic idempotent; re-running a scan for the same window must not duplicate signals (MERGE on entity and window).

## References
- references/cost-usage-anomaly-detection.md: credit spike detection, baselines, ML anomaly detection, budgets, and resource monitors for early overrun warning.
- references/cost-warehouse-efficiency-analysis.md: idle warehouse detection, auto-suspend tuning, oversizing and undersizing, multi-cluster settings.
- references/cost-query-efficiency-analysis.md: costly and inefficient query patterns, spilling, pruning, queuing, and per-user or per-role cost attribution.
- references/cost-savings-impact-modeling.md: Snowpark procedures that estimate savings and performance risk for resize, suspend, and query changes.
- references/cost-recommendation-handoff.md: ranking logic, Operations Lead review, approve, adjust, decline flow, controlled application, and rollback.
- references/cost-analyst-and-agent-interface.md: Cortex Analyst semantic view for credit trends and the Cortex Agent definition that ties the tools together.