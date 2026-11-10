---
name: cost-analyst-and-agent-interface
description: This skill defines the conversational interface of the Cost Optimization Agent, building a Cortex Analyst semantic view over credit, warehouse, and query attribution data and a Cortex Agent that combines text-to-SQL cost questions with tools for reading the review queue and triggering savings models. Use this skill for any task involving natural language cost questions, credit trend exploration, semantic view or semantic model design for FinOps, Cortex Agent configuration, Snowflake Intelligence cost assistants, or answering questions such as which team drives spend or why credits rose last week.
---

# Cost Analyst and Agent Interface

## Overview
Cortex Analyst surfaces credit trends and cost drivers through governed natural language queries. A Cortex Agent orchestrates that analyst tool with custom tools for the recommendation queue, giving FinOps teams, platform owners, and budget holders one conversational entry point.

## When to Use
- Letting finance users ask cost questions in plain language.
- Defining metrics such as total credits, idle credits, and attributed credits.
- Creating or updating the Cortex Agent for cost optimization.

## Instructions
1. Materialize a daily attribution fact table refreshed by the cost task graph.

        CREATE OR REPLACE TABLE COST_OPS_DB.FINOPS.FACT_DAILY_QUERY_COST AS
        SELECT DATE_TRUNC('DAY', a.START_TIME)::DATE AS USAGE_DATE,
               a.WAREHOUSE_NAME, a.USER_NAME, q.ROLE_NAME,
               COALESCE(NULLIF(q.QUERY_TAG, ''), 'UNTAGGED') AS QUERY_TAG,
               SUM(a.CREDITS_ATTRIBUTED_COMPUTE) AS ATTRIBUTED_CREDITS,
               COUNT(*) AS QUERY_COUNT
        FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY a
        JOIN SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY q ON q.QUERY_ID = a.QUERY_ID
        WHERE a.START_TIME >= DATEADD('DAY', -90, CURRENT_DATE())
        GROUP BY 1, 2, 3, 4, 5;

2. Create the semantic view.

        CREATE OR REPLACE SEMANTIC VIEW COST_OPS_DB.FINOPS.SV_CREDIT_COST
            TABLES (
                wh AS COST_OPS_DB.FINOPS.V_DAILY_WAREHOUSE_CREDITS
                    PRIMARY KEY (WAREHOUSE_NAME, USAGE_DATE)
                    COMMENT = 'Daily metered credits per warehouse',
                qc AS COST_OPS_DB.FINOPS.FACT_DAILY_QUERY_COST
                    COMMENT = 'Daily compute credits attributed to users, roles, and query tags',
                rec AS COST_OPS_DB.FINOPS.COST_RECOMMENDATIONS
                    PRIMARY KEY (REC_ID)
                    COMMENT = 'Savings recommendations and their decision status'
            )
            RELATIONSHIPS (
                qc_to_wh AS qc (WAREHOUSE_NAME, USAGE_DATE) REFERENCES wh
            )
            FACTS (
                wh.total_credits_fact AS TOTAL_CREDITS,
                wh.cloud_services_fact AS CLOUD_SERVICES_CREDITS,
                qc.attributed_credits_fact AS ATTRIBUTED_CREDITS,
                rec.savings_fact AS EST_MONTHLY_SAVINGS_CR
            )
            DIMENSIONS (
                wh.warehouse_name AS WAREHOUSE_NAME WITH SYNONYMS = ('warehouse', 'compute cluster'),
                wh.usage_date AS USAGE_DATE WITH SYNONYMS = ('date', 'day'),
                qc.user_name AS USER_NAME WITH SYNONYMS = ('user', 'who'),
                qc.role_name AS ROLE_NAME WITH SYNONYMS = ('role', 'team role'),
                qc.query_tag AS QUERY_TAG WITH SYNONYMS = ('team', 'application', 'workload'),
                rec.action_type AS ACTION_TYPE,
                rec.status AS STATUS
            )
            METRICS (
                wh.total_credits AS SUM(wh.total_credits_fact)
                    COMMENT = 'Total credits including cloud services',
                qc.attributed_credits AS SUM(qc.attributed_credits_fact)
                    COMMENT = 'Compute credits attributed to queries, excluding idle time',
                rec.pending_savings AS SUM(IFF(rec.status = 'PENDING', rec.savings_fact, 0))
                    COMMENT = 'Estimated monthly credits saved if pending actions are approved'
            )
            COMMENT = 'FinOps semantic view for credit trends, cost drivers, and savings pipeline';

        GRANT SELECT ON SEMANTIC VIEW COST_OPS_DB.FINOPS.SV_CREDIT_COST TO ROLE FINOPS_AGENT_ROLE;

3. Create the Cortex Agent with the analyst tool and a procedure tool for the review queue.

        CREATE OR REPLACE PROCEDURE COST_OPS_DB.FINOPS.SP_GET_REVIEW_QUEUE(P_LIMIT NUMBER)
        RETURNS TABLE()
        LANGUAGE SQL
        AS
        $$
        DECLARE res RESULTSET DEFAULT (
            SELECT * FROM COST_OPS_DB.FINOPS.V_REVIEW_QUEUE ORDER BY PRIORITY_RANK LIMIT :P_LIMIT);
        BEGIN RETURN TABLE(res); END;
        $$;

        CREATE OR REPLACE AGENT COST_OPS_DB.FINOPS.COST_OPTIMIZATION_AGENT
            COMMENT = 'Monitors Snowflake credit consumption and proposes ranked savings actions'
            FROM SPECIFICATION
            $$
            models:
              orchestration: auto
            instructions:
              orchestration: >
                Use CreditAnalyst for any question about credits, trends, warehouses, users, roles, or teams.
                Use ReviewQueue when the user asks what to optimize, which actions are pending, or expected savings.
                Never claim a change was applied; changes require Operations Lead approval.
              response: >
                Lead with the number, then the driver. Report credits and state the time window.
                Present recommendations ranked, with estimated monthly savings, risk, and confidence.
            tools:
              - tool_spec:
                  type: cortex_analyst_text_to_sql
                  name: CreditAnalyst
                  description: Answers questions about Snowflake credit usage, trends, and cost attribution.
              - tool_spec:
                  type: generic
                  name: ReviewQueue
                  description: Returns the top pending cost optimization recommendations ranked by priority.
                  input_schema:
                    type: object
                    properties:
                      p_limit:
                        type: number
                        description: Number of recommendations to return
                    required: [p_limit]
            tool_resources:
              CreditAnalyst:
                semantic_view: COST_OPS_DB.FINOPS.SV_CREDIT_COST
                execution_environment:
                  type: warehouse
                  warehouse: FINOPS_WH
              ReviewQueue:
                type: procedure
                identifier: COST_OPS_DB.FINOPS.SP_GET_REVIEW_QUEUE
                execution_environment:
                  type: warehouse
                  warehouse: FINOPS_WH
            $$;

        GRANT USAGE ON AGENT COST_OPS_DB.FINOPS.COST_OPTIMIZATION_AGENT TO ROLE FINOPS_APPROVER_ROLE;

4. Seed verified queries for common prompts: credits by warehouse last 7 days, top 10 users by attributed credits this month, week-over-week change by query tag, and pending savings by action type.

## Coding Conventions
- Every dimension and metric carries synonyms or comments that match FinOps vocabulary.
- Expose no write tools to the agent; approvals stay in SP_DECIDE_RECOMMENDATION.
- Keep the semantic view on curated objects in COST_OPS_DB.FINOPS, never directly on ACCOUNT_USAGE.
- Version the agent specification in source control alongside the semantic view DDL.