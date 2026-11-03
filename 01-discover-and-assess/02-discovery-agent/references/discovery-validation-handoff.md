---
name: discovery-validation-handoff
description: This skill defines how a Discovery Agent packages inventories, dependency maps, flagged risks, and unresolved findings for Lead Validator review, then records decisions and releases confirmed scope through Snowflake tables, Snowflake Horizon, and Cortex Agents. Use this skill for any task involving discovery reports, validator review packages, scope confirmation, disputed findings, decision logs, audit trails, or handoff to the Data Assessment Agent, including migration readiness summaries, exception queues, and approved source lists.
---

# Discovery Validation Handoff

## Overview

This skill covers the Hand Off phase of the Discovery Agent. It gives the Lead Validator the evidence needed to confirm scope and resolve ambiguous or disputed findings.

## When to Use

- Prepare a discovery review package.
- Summarize assets, dependencies, and flagged risks.
- Record scope confirmation or rejection.
- Resolve disputed discovery findings.
- Release confirmed sources to the Data Assessment Agent.

## Instructions

1. Assemble the package only after all source phases are complete or explicitly marked partial.
2. Include the following sections:
    - Sweep coverage and limitations
    - New, changed, removed, and unverified assets
    - Inventory summary by source and asset type
    - Upstream and downstream dependency map
    - Undocumented, orphaned, and unresolved components
    - Findings that require a human decision
3. Rank findings by business impact, dependency reach, and confidence.
4. Present evidence and recommendations without making the final decision.
5. Require a decision and rationale from the Lead Validator for each finding.
6. Treat decisions as append-only so the audit trail remains intact.
7. Release only `CONFIRMED` assets to the Data Assessment Agent.
8. Reopen a decision when a later sweep materially changes the affected asset or dependency.

Create the decision log:

    CREATE TABLE IF NOT EXISTS DISCOVERY_DB.VALIDATION.DISCOVERY_DECISION (
        DECISION_ID         STRING NOT NULL,
        SWEEP_ID            STRING NOT NULL,
        ASSET_ID            STRING,
        FINDING_TYPE        STRING NOT NULL,
        AGENT_SUMMARY       STRING NOT NULL,
        EVIDENCE            VARIANT,
        DECISION_STATUS     STRING NOT NULL,
        DECISION_RATIONALE  STRING,
        DECIDED_BY          STRING,
        DECIDED_AT          TIMESTAMP_NTZ,
        CONSTRAINT PK_DISCOVERY_DECISION PRIMARY KEY (DECISION_ID)
    );

Use these decision values: `PENDING`, `CONFIRMED`, `REJECTED`, `DISPUTED`, and `NEEDS_MORE_EVIDENCE`.

Expose the latest confirmed scope for the Data Assessment Agent:

    CREATE OR REPLACE VIEW DISCOVERY_DB.VALIDATION.V_CONFIRMED_SCOPE AS
    SELECT
        A.ASSET_ID,
        A.SOURCE_SYSTEM_ID,
        A.ASSET_TYPE,
        A.ASSET_QUALIFIED_NAME,
        D.SWEEP_ID,
        D.DECIDED_BY,
        D.DECIDED_AT
    FROM DISCOVERY_DB.CATALOG.DISCOVERED_ASSET A
    JOIN DISCOVERY_DB.VALIDATION.DISCOVERY_DECISION D
        ON A.ASSET_ID = D.ASSET_ID
    WHERE D.FINDING_TYPE = 'SCOPE'
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY D.ASSET_ID
        ORDER BY D.DECIDED_AT DESC NULLS LAST
    ) = 1
        AND D.DECISION_STATUS = 'CONFIRMED';

## Coding Conventions

- Never auto-confirm scope or auto-resolve disputed findings.
- Link every summary statement to catalog, dependency, or sweep evidence.
- Keep agent recommendations separate from human decisions.
- Use controlled values for finding types and decision statuses.
- Preserve full decision history, including reversals.
- Clearly label partial coverage and unresolved findings.