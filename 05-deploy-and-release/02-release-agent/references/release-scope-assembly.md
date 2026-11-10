---
name: release-scope-assembly
description: This skill defines how the Release Agent detects changes that passed testing, groups them into a versioned release scope, and validates object dependencies, ordering, and conflicts using streams on test results, ACCOUNT_USAGE.OBJECT_DEPENDENCIES, SNOWFLAKE.CORE.GET_LINEAGE, and Git repository metadata. Use this skill for any task involving release scope definition, release candidate selection, dependency checks, deployment ordering, change conflict detection, blocked-change reporting, or prompts such as build the release manifest, what is ready to ship, or check dependencies before go-live.
---
# Release Scope Assembly

## Overview
Scope assembly is the Sense and first Act phase of the Release Agent. It selects eligible changes, builds a manifest, and proves the release can deploy safely by checking upstream and downstream dependencies, deployment order, and conflicting changes.

## When to Use
- Selecting changes for the next release window.
- Generating or regenerating a release manifest.
- Checking whether a change depends on objects not yet released.
- Identifying downstream objects affected by a DROP or ALTER.
- Rebuilding scope after a Release Gatekeeper modify request.

## Instructions

1. Select eligible candidates: test PASSED, not assigned to a release, latest commit per object.

    CREATE OR REPLACE VIEW RELEASE_DB.CORE.V_ELIGIBLE_CHANGES AS
    SELECT *
    FROM RELEASE_DB.CORE.RELEASE_CANDIDATES
    WHERE TEST_STATUS = 'PASSED'
      AND RELEASE_ID IS NULL
    QUALIFY ROW_NUMBER() OVER (PARTITION BY OBJECT_FQN ORDER BY QUEUED_AT DESC) = 1;

2. Create the package and assign changes in one transaction.

    CREATE OR REPLACE PROCEDURE RELEASE_DB.CORE.SP_ASSEMBLE_SCOPE(TARGET_ENV STRING)
    RETURNS STRING
    LANGUAGE SQL
    EXECUTE AS OWNER
    AS
    $$
    DECLARE
        rel_id STRING;
        cnt INTEGER;
    BEGIN
        SELECT COUNT(*) INTO :cnt FROM RELEASE_DB.CORE.V_ELIGIBLE_CHANGES;
        IF (cnt = 0) THEN
            RETURN 'NO_ELIGIBLE_CHANGES';
        END IF;

        rel_id := 'REL-' || :TARGET_ENV || '-' || TO_CHAR(CURRENT_DATE(), 'YYYYMMDD') || '-' ||
                  LPAD((SELECT COUNT(*) + 1 FROM RELEASE_DB.CORE.RELEASE_PACKAGES
                        WHERE TO_DATE(CREATED_AT) = CURRENT_DATE() AND TARGET_ENV = :TARGET_ENV)::STRING, 3, '0');

        BEGIN TRANSACTION;
        UPDATE RELEASE_DB.CORE.RELEASE_CANDIDATES c
            SET RELEASE_ID = :rel_id
            FROM RELEASE_DB.CORE.V_ELIGIBLE_CHANGES e
            WHERE c.CHANGE_ID = e.CHANGE_ID;

        INSERT INTO RELEASE_DB.CORE.RELEASE_PACKAGES (RELEASE_ID, TARGET_ENV, STATUS, MANIFEST)
        SELECT :rel_id, :TARGET_ENV, 'DRAFT',
               OBJECT_CONSTRUCT(
                   'change_count', COUNT(*),
                   'changes', ARRAY_AGG(OBJECT_CONSTRUCT(
                       'change_id', CHANGE_ID, 'object', OBJECT_FQN, 'type', CHANGE_TYPE,
                       'commit', COMMIT_SHA, 'ticket', TICKET_ID, 'test_run', TEST_RUN_ID)))
        FROM RELEASE_DB.CORE.RELEASE_CANDIDATES
        WHERE RELEASE_ID = :rel_id;
        COMMIT;

        RETURN rel_id;
    END;
    $$;

3. Check upstream dependencies: every referenced object must exist in the target environment or be in the same release.

    SELECT d.REFERENCING_OBJECT_NAME, d.REFERENCED_DATABASE || '.' || d.REFERENCED_SCHEMA || '.' || d.REFERENCED_OBJECT_NAME AS REFERENCED_FQN
    FROM SNOWFLAKE.ACCOUNT_USAGE.OBJECT_DEPENDENCIES d
    JOIN RELEASE_DB.CORE.RELEASE_CANDIDATES c
      ON d.REFERENCING_DATABASE || '.' || d.REFERENCING_SCHEMA || '.' || d.REFERENCING_OBJECT_NAME = c.OBJECT_FQN
    WHERE c.RELEASE_ID = :rel_id;

    Classify each unmatched reference as BLOCKING (missing in target, not in scope) or SATISFIED_IN_SCOPE.

4. Check downstream impact with lineage, especially for DROP and ALTER.

    SELECT *
    FROM TABLE(SNOWFLAKE.CORE.GET_LINEAGE('ANALYTICS_DB.SALES.ORDERS', 'TABLE', 'DOWNSTREAM', 3));

    Flag downstream objects outside the release as IMPACTED and include them in DEPENDENCY_REPORT for gatekeeper review.

5. Detect conflicts:
    - Two changes touching the same OBJECT_FQN across open releases.
    - A DROP of an object referenced by another in-scope change.
    - Changes whose commits are not on the release branch in the Snowflake Git repository.

    ALTER GIT REPOSITORY RELEASE_DB.CORE.APP_REPO FETCH;
    SHOW GIT BRANCHES IN RELEASE_DB.CORE.APP_REPO;

6. Compute deployment order with a topological sort of in-scope dependencies (Python procedure using networkx or plain Kahn's algorithm). Default ordering by type when no edges exist: DATABASE, SCHEMA, SEQUENCE, TABLE, VIEW, FUNCTION, PROCEDURE, STREAM, TASK, DATA.

7. Persist results.

    UPDATE RELEASE_DB.CORE.RELEASE_PACKAGES
    SET DEPENDENCY_REPORT = :report,
        STATUS = IFF(:blocking_count = 0, 'READY', 'DRAFT'),
        UPDATED_AT = CURRENT_TIMESTAMP()
    WHERE RELEASE_ID = :rel_id;

Best practices:
- ACCOUNT_USAGE views have latency of up to a few hours; for same-day objects, supplement with INFORMATION_SCHEMA and parsed DDL from the repository.
- Treat any unresolved BLOCKING dependency as a hard stop.
- Remove blocked changes from scope (set RELEASE_ID to NULL) rather than shipping a partial dependency chain.

Output expectations: a manifest VARIANT and a DEPENDENCY_REPORT shaped as:

    {
        "blocking": [{"change_id": "...", "missing": "DB.SCH.OBJ"}],
        "impacted_downstream": ["DB.SCH.VIEW_A"],
        "conflicts": [],
        "deploy_order": ["CHG-101", "CHG-104", "CHG-102"],
        "checked_at": "2026-11-10T12:00:00Z"
    }

## Coding Conventions
- CHANGE_ID format: CHG-<TICKET>-<SHORT_SHA>.
- Always store fully qualified object names as DATABASE.SCHEMA.OBJECT in uppercase.
- Wrap scope assignment in an explicit transaction.
- Keep dependency logic in SP_CHECK_DEPENDENCIES, separate from SP_ASSEMBLE_SCOPE.
- Report keys in snake_case; arrays sorted by deployment order.