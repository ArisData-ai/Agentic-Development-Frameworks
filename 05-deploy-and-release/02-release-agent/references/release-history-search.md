---
name: release-history-search
description: This skill defines how the Release Agent indexes past release packages, notes, and audit records into a Cortex Search service so that agents and users can retrieve prior releases, similar changes, and historical decisions with filtered semantic search. Use this skill for any task involving release history lookup, finding previous releases that touched an object, reusing past release notes, incident root-cause research by release, Cortex Search service creation or refresh, or connecting release history as a search tool for Cortex Agents.
---
# Release History Search

## Overview
Cortex Search makes the release record searchable in natural language. Release managers find what shipped and when, incident responders trace which release touched an object, and the Release Agent retrieves prior notes for consistent wording and precedent decisions.

## When to Use
- Creating or refreshing the release history search service.
- Answering questions like which release changed a given table, or why a release was deferred.
- Adding release history as a tool on the Release Agent or other pipeline agents.
- Supporting the Remediation and Observability agents with change context during incidents.

## Instructions

1. Build a flattened, search-ready source view with one row per change per release.

    CREATE OR REPLACE VIEW RELEASE_DB.SEARCH.V_RELEASE_DOCS AS
    SELECT
        p.RELEASE_ID,
        p.RELEASE_VERSION,
        p.TARGET_ENV,
        p.STATUS,
        f.value:change_id::STRING  AS CHANGE_ID,
        f.value:object::STRING     AS OBJECT_FQN,
        f.value:ticket::STRING     AS TICKET_ID,
        f.value:type::STRING       AS CHANGE_TYPE,
        p.CREATED_AT::DATE         AS RELEASE_DATE,
        'Release ' || p.RELEASE_ID || ' (' || COALESCE(p.RELEASE_VERSION, 'unversioned') || ') to ' || p.TARGET_ENV ||
        '. Change ' || f.value:change_id::STRING || ' ' || f.value:type::STRING || ' on ' || f.value:object::STRING ||
        '. Notes: ' || COALESCE(p.RELEASE_NOTES:ENGINEERING::STRING, '') ||
        ' Decision history: ' || COALESCE(d.DECISIONS, '') AS SEARCH_TEXT
    FROM RELEASE_DB.CORE.RELEASE_PACKAGES p,
         LATERAL FLATTEN(p.MANIFEST:changes) f
    LEFT JOIN (
        SELECT RELEASE_ID, LISTAGG(DECISION || ': ' || COALESCE(REASON, ''), '; ') AS DECISIONS
        FROM RELEASE_DB.CORE.GATEKEEPER_DECISIONS GROUP BY RELEASE_ID
    ) d ON d.RELEASE_ID = p.RELEASE_ID;

2. Create the Cortex Search service.

    CREATE OR REPLACE CORTEX SEARCH SERVICE RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC
        ON SEARCH_TEXT
        ATTRIBUTES RELEASE_ID, TARGET_ENV, STATUS, OBJECT_FQN, TICKET_ID, CHANGE_TYPE, RELEASE_DATE
        WAREHOUSE = RELEASE_WH
        TARGET_LAG = '1 hour'
        EMBEDDING_MODEL = 'snowflake-arctic-embed-l-v2.0'
    AS
        SELECT * FROM RELEASE_DB.SEARCH.V_RELEASE_DOCS;

    GRANT USAGE ON CORTEX SEARCH SERVICE RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC TO ROLE RELEASE_AGENT_ROLE;
    GRANT USAGE ON CORTEX SEARCH SERVICE RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC TO ROLE RELEASE_AUDITOR_ROLE;

3. Query with filters from SQL for testing.

    SELECT PARSE_JSON(
        SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
            'RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC',
            '{
                "query": "changes to the orders table that were deferred",
                "columns": ["RELEASE_ID", "OBJECT_FQN", "STATUS", "SEARCH_TEXT"],
                "filter": {"@eq": {"TARGET_ENV": "PROD"}},
                "limit": 5
            }'))['results'] AS RESULTS;

4. Query from Python in procedures or Streamlit apps.

    from snowflake.core import Root
    root = Root(session)
    svc = (root.databases["RELEASE_DB"].schemas["SEARCH"]
               .cortex_search_services["RELEASE_HISTORY_SVC"])
    resp = svc.search(
        query="previous releases touching ANALYTICS_DB.SALES.ORDERS",
        columns=["RELEASE_ID", "RELEASE_DATE", "SEARCH_TEXT"],
        filter={"@eq": {"OBJECT_FQN": "ANALYTICS_DB.SALES.ORDERS"}},
        limit=10)

5. Register the service as the release_history tool in the Release Agent specification (see release-agent) and set max_results between 5 and 10.

6. Monitor freshness and cost.

    SHOW CORTEX SEARCH SERVICES IN SCHEMA RELEASE_DB.SEARCH;
    DESCRIBE CORTEX SEARCH SERVICE RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC;

    Suspend indexing in non-production environments when not in use:

    ALTER CORTEX SEARCH SERVICE RELEASE_DB.SEARCH.RELEASE_HISTORY_SVC SUSPEND INDEXING;

Best practices:
- Index only finalized information; exclude DRAFT packages if early drafts cause noisy results.
- Use exact-match filters on OBJECT_FQN and TICKET_ID for precise lookups, and semantic query text for intent.
- Keep SEARCH_TEXT under a few thousand tokens per row; split long notes by section if needed.
- Do not index COMPLIANCE notes or sensitive DETAILS in a service granted to broad roles.

Output expectations: a running RELEASE_HISTORY_SVC with an acceptable target lag, filterable attributes, and agent and app integrations that return ranked release records with RELEASE_ID citations.

## Coding Conventions
- Search services suffixed _SVC and placed in RELEASE_DB.SEARCH.
- Source views prefixed V_ and kept separate from core tables.
- Attribute columns in uppercase and limited to filterable, low-cardinality or exact-match fields.
- Pin EMBEDDING_MODEL explicitly; changing it requires a service rebuild and a logged change.
- Always return RELEASE_ID in result columns for traceability.