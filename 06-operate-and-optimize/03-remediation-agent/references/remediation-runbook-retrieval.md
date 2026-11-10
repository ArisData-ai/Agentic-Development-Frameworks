---
name: remediation-runbook-retrieval
description: This skill defines runbook and incident history retrieval for the Remediation Agent, which indexes runbooks, past incident resolutions, and error code guides into a Cortex Search service and returns ranked, filtered passages as diagnostic evidence. Use this skill for any task involving creating a Cortex Search service for runbooks, ingesting Confluence, ServiceNow, Jira, or Markdown runbooks, searching past incident fixes, filtering by error code or object type, or grounding remediation proposals in documented procedures.
---
# Remediation Runbook Retrieval

## Overview
Retrieval grounds every proposal in documented knowledge. A Cortex Search service indexes runbook steps and resolved incidents so the diagnosis agent cites evidence rather than guessing.

## When to Use
- Building or refreshing the runbook corpus
- Adding past incident resolutions as searchable knowledge
- Tuning retrieval filters and result counts

## Instructions
1. Create a corpus table that unifies runbooks and resolved incidents.

        CREATE TABLE IF NOT EXISTS OPS_DB.REMEDIATION.KNOWLEDGE_CORPUS (
            doc_id        STRING,
            doc_type      STRING,     -- RUNBOOK | RESOLVED_INCIDENT | ERROR_GUIDE
            source_system STRING,     -- CONFLUENCE | SERVICENOW | JIRA | GIT
            object_type   STRING,     -- TASK | PIPE | STREAM | DYNAMIC_TABLE
            error_code    STRING,
            title         STRING,
            content       STRING,
            updated_at    TIMESTAMP_LTZ
        );

2. Chunk long runbooks into step-level passages of roughly 300 to 500 tokens before loading.
3. Create the Cortex Search service.

        CREATE OR REPLACE CORTEX SEARCH SERVICE OPS_DB.REMEDIATION.RUNBOOK_SEARCH
            ON content
            ATTRIBUTES doc_type, object_type, error_code, source_system
            WAREHOUSE = REMEDIATION_WH
            TARGET_LAG = '1 hour'
            AS (
                SELECT doc_id, doc_type, object_type, error_code,
                       source_system, title, content
                FROM OPS_DB.REMEDIATION.KNOWLEDGE_CORPUS
            );

4. Query with attribute filters scoped to the incident.

        SELECT PARSE_JSON(SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
            'OPS_DB.REMEDIATION.RUNBOOK_SEARCH',
            '{
                "query": "task failed warehouse suspended insufficient credits",
                "columns": ["doc_id", "title", "content", "doc_type"],
                "filter": {"@eq": {"object_type": "TASK"}},
                "limit": 5
            }'))['results'] AS results;

5. Append every approved and successful resolution back into KNOWLEDGE_CORPUS as RESOLVED_INCIDENT so retrieval improves over time.

## Coding Conventions
- Keep one search service per environment; filter by attributes rather than creating per-team services.
- Never index secrets, credentials, or connection strings in runbook content.
- Return doc_id with every result so proposals can cite sources.
- Grant USAGE on the search service only to REMEDIATION_AGENT_ROLE and OPERATIONS_LEAD_ROLE.