---
name: remediation-root-cause-diagnosis
description: This skill defines root cause diagnosis for the Remediation Agent, which plans investigation with a Cortex Agent over incident context, query history, and runbook search results and outputs a structured proposal containing root cause, confidence, fix, and risk tier into REMEDIATION_PROPOSALS. Use this skill for any task involving configuring a Cortex Agent for incident diagnosis, writing orchestration instructions, classifying errors, scoring confidence, drafting fixes or runbook steps, assigning SAFE or REVIEW risk tiers, or producing JSON remediation proposals.
---
# Remediation Root Cause Diagnosis

## Overview
Diagnosis turns an incident row into an evidence-backed proposal. A Cortex Agent combines telemetry tools and runbook retrieval, then emits a strict JSON proposal that downstream skills validate and route.

## When to Use
- Creating or updating the diagnosis agent specification
- Changing proposal schema, confidence rules, or risk tier logic
- Debugging low-quality or ungrounded proposals

## Instructions
1. Create the Cortex Agent with runbook search as a tool.

        CREATE OR REPLACE AGENT OPS_DB.REMEDIATION.REMEDIATION_AGENT
            COMMENT = 'Diagnoses incidents and proposes remediations'
            FROM SPECIFICATION
            $$
            models:
              orchestration: claude-4-sonnet
            instructions:
              orchestration: >
                Diagnose the incident using its error code, error message, and context.
                Always search runbooks and past incidents before proposing a fix.
                Classify the root cause as one of: TRANSIENT, RESOURCE, PERMISSION,
                SCHEMA_DRIFT, UPSTREAM_DATA, CODE_DEFECT, UNKNOWN.
              response: >
                Return only JSON with keys root_cause, category, confidence,
                action_type, action_sql, risk_tier, evidence_doc_ids, rationale.
            tools:
              - tool_spec:
                  type: "cortex_search"
                  name: "runbook_search"
            tool_resources:
              runbook_search:
                name: "OPS_DB.REMEDIATION.RUNBOOK_SEARCH"
                max_results: 5
            $$;

2. Enrich incident context before calling the agent: failing query text, warehouse state, recent DDL on referenced objects, and upstream task status.

        SELECT query_text, error_code, error_message, warehouse_name, execution_status
        FROM SNOWFLAKE.ACCOUNT_USAGE.QUERY_HISTORY
        WHERE query_id = :query_id;

3. Apply deterministic risk tiering after the agent responds; the agent suggests, the rules decide.

        risk_tier = SAFE    if action_type IN allowlist AND confidence >= 0.80
        risk_tier = REVIEW  if action_type = CUSTOM_SQL OR confidence < 0.80
        risk_tier = BLOCKED if action_sql contains DROP | TRUNCATE | DELETE | GRANT | REVOKE

4. Validate the JSON, then insert into REMEDIATION_PROPOSALS with evidence_doc_ids stored in evidence.
5. Mark the incident DIAGNOSED; set UNKNOWN causes to REVIEW with a runbook step rather than SQL.

## Coding Conventions
- Require evidence_doc_ids for any confidence above 0.5; reject proposals without citations.
- Never let model output bypass the rule-based tiering step.
- Keep action_sql to a single statement; multi-step fixes become ordered runbook steps.
- Log the full agent request and response in evidence for audit.