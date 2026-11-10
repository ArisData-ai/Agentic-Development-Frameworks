---
name: ci-cd-change-detection
description: This skill defines how the CI/CD Agent detects merged commits and new build artifacts, using a Snowflake Git repository object, SHOW GIT BRANCHES commit hashes, and Streams on stage directory tables to create DETECTED pipeline runs in the control schema. Use this skill for any task involving connecting GitHub, GitLab, Bitbucket, or Azure Repos to Snowflake, polling branches for new commits, detecting new Snowpark wheels or SQL bundles on stage, triggering a pipeline on merge, or deduplicating already-processed commits.
---

# CI/CD Change Detection

## Overview

Change detection is the sensing stage of the CI/CD Agent. It finds two kinds of signal and turns each new one into a single row in CICD_DB.CONTROL.PIPELINE_RUNS with STATUS = 'DETECTED':

- Merged commits on tracked branches, read through a Snowflake Git repository object.
- New build artifacts uploaded to an internal stage, such as Snowpark packages or rendered SQL bundles.

## When to Use

- You are connecting a source repository to Snowflake for the first time.
- A pipeline needs to start automatically on merge to main or a release branch.
- An external CI runner uploads artifacts and Snowflake needs to react to them.
- The same commit is being processed more than once and needs deduplication.

## Instructions

1. Store Git credentials and create the API integration.

        CREATE OR REPLACE SECRET CICD_DB.SECRETS.GIT_PAT
            TYPE = PASSWORD
            USERNAME = '<git_service_user>'
            PASSWORD = '<personal_access_token>';

        CREATE OR REPLACE API INTEGRATION GIT_API_INT
            API_PROVIDER = git_https_api
            API_ALLOWED_PREFIXES = ('https://github.com/<org>/')
            ALLOWED_AUTHENTICATION_SECRETS = (CICD_DB.SECRETS.GIT_PAT)
            ENABLED = TRUE;

2. Create the repository object.

        CREATE OR REPLACE GIT REPOSITORY CICD_DB.SRC.DATA_REPO
            API_INTEGRATION = GIT_API_INT
            GIT_CREDENTIALS = CICD_DB.SECRETS.GIT_PAT
            ORIGIN = 'https://github.com/<org>/<repo>.git';

3. Create the commit detection procedure. It fetches, compares the branch head with the last processed hash, and inserts a run only when the commit is new.

        CREATE OR REPLACE PROCEDURE CICD_DB.ORCH.SP_DETECT_CHANGES(P_BRANCH STRING)
        RETURNS STRING
        LANGUAGE SQL
        EXECUTE AS OWNER
        AS
        $$
        DECLARE
            v_head STRING;
            v_last STRING;
        BEGIN
            ALTER GIT REPOSITORY CICD_DB.SRC.DATA_REPO FETCH;
            SHOW GIT BRANCHES IN CICD_DB.SRC.DATA_REPO;
            SELECT "commit_hash" INTO :v_head
                FROM TABLE(RESULT_SCAN(LAST_QUERY_ID()))
                WHERE "name" = :P_BRANCH;

            SELECT MAX_BY(COMMIT_HASH, CREATED_AT) INTO :v_last
                FROM CICD_DB.CONTROL.PIPELINE_RUNS
                WHERE BRANCH = :P_BRANCH;

            IF (v_head IS NULL OR v_head = COALESCE(v_last, '')) THEN
                RETURN 'NO_CHANGE';
            END IF;

            INSERT INTO CICD_DB.CONTROL.PIPELINE_RUNS (BRANCH, COMMIT_HASH, TARGET_ENV, STATUS)
                VALUES (:P_BRANCH, :v_head, 'DEV', 'DETECTED');
            RETURN 'DETECTED:' || v_head;
        END;
        $$;

4. Detect artifact uploads with a directory table stream.

        CREATE STAGE IF NOT EXISTS CICD_DB.ARTIFACTS.BUILD_STAGE
            DIRECTORY = (ENABLE = TRUE)
            ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE');

        CREATE STREAM IF NOT EXISTS CICD_DB.ARTIFACTS.BUILD_STAGE_STREAM
            ON STAGE CICD_DB.ARTIFACTS.BUILD_STAGE;

        -- CI runner, after upload:
        --     snow stage copy dist/ @CICD_DB.ARTIFACTS.BUILD_STAGE/<commit_hash>/ --recursive
        --     snow sql -q "ALTER STAGE CICD_DB.ARTIFACTS.BUILD_STAGE REFRESH"

        CREATE TABLE IF NOT EXISTS CICD_DB.CONTROL.ARTIFACT_EVENTS (
            RELATIVE_PATH STRING, SIZE NUMBER, MD5 STRING, EVENT_TS TIMESTAMP_LTZ
        );

        INSERT INTO CICD_DB.CONTROL.ARTIFACT_EVENTS
            SELECT RELATIVE_PATH, SIZE, MD5, CURRENT_TIMESTAMP()
            FROM CICD_DB.ARTIFACTS.BUILD_STAGE_STREAM
            WHERE METADATA$ACTION = 'INSERT';

5. Hand the DETECTED run to the build stage through the task graph root, as defined in ci-cd-orchestration-and-failure-handling.

## Coding Conventions

- Track branches explicitly. Never detect on wildcard branches in production pipelines.
- Artifact paths follow @BUILD_STAGE/<commit_hash>/<artifact_name> so every artifact traces back to one commit.
- Always consume streams inside a DML statement so offsets advance exactly once.
- Detection procedures are read-mostly and must never deploy objects.
- Rotate GIT_PAT through ALTER SECRET. Do not recreate integrations to rotate credentials.