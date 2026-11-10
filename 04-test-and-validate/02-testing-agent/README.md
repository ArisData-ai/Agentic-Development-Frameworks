# Testing Agent

*Segment 4: Test & Validate | AI Agents for the Whole Project Pipeline*

The Testing Agent generates and runs tests that cover the edge cases teams often miss. It watches for new or changed code, pipelines, and models. When it finds one, it generates edge-case and regression tests, runs them inside Snowflake, and logs defects with their likely causes. A **QA Lead** then accepts, reprioritizes, or rejects every finding.

The goal is broader test coverage without a matching increase in manual effort.

---

## How It Works

```
SENSE     Detect new or changed tables, views, dynamic tables, procedures, pipelines
   │      ──> TESTS.CHANGE_EVENTS (PENDING)
   ▼
ACT       Generate edge-case and regression tests
   │      ──> TESTS.TEST_CASES (PROPOSED | ACTIVE)
   │      Run tests in Snowpark
   │      ──> RESULTS.TEST_RUNS / RESULTS.TEST_RESULTS
   │      Log defects with likely causes
   │      ──> DEFECTS.DEFECT_LOG (OPEN)
   ▼
HAND OFF  QA Lead reviews DEFECTS.V_QA_REVIEW_QUEUE
          ──> ACCEPTED | REPRIORITIZED | REJECTED
```

The agent proposes and the QA Lead decides. The agent never writes to target objects, never closes defects, and never adds a test to the permanent regression suite without QA Lead approval.

---

## Snowflake Components

| Component | Role in the agent |
|---|---|
| Cortex Agents | Plans which test suites to run for each change and coordinates the tools |
| Snowpark | Runs test logic, fixtures, and comparisons inside the platform |
| Streams & Tasks | Trigger the loop on change and on a schedule |
| Cortex COMPLETE | Proposes logic-specific edge-case tests and likely-cause hypotheses |
| Cortex Analyst | Answers questions about pass rates and defect trends |
| Time Travel and zero-copy clones | Provide regression baselines and isolated fixture environments |

---

## Skill Files

```
02-testing-agent/
├── SKILL.md                                     # Primary: operating model, setup, task graph, guardrails
├── references
        ├── testing-agent-change-detection.md    # Sense: scope, DDL detection, CI registration, impact expansion
        ├── testing-agent-test-generation.md     # Act: test contract, edge-case templates, Cortex tests, baselines
        ├── testing-agent-test-execution.md      # Act: Snowpark runner, clone fixtures, retries, results
        └── testing-agent-defect-reporting.md    # Hand off: dedup, severity, likely cause, QA queue, metrics
```

Start with `SKILL`. It links to the sibling skills for each phase.

---

## Prerequisites

- A Snowflake account with Cortex AI functions enabled in your region, or with cross-region inference turned on
- The `SYSADMIN` and `SECURITYADMIN` roles, or equivalents, to provision objects and roles
- Access to `SNOWFLAKE.ACCOUNT_USAGE` through `IMPORTED PRIVILEGES`
- The `SNOWFLAKE.CORTEX_USER` database role
- Read access to the target databases under test
- Optional: an email notification integration for QA alerts
- Optional: a CI/CD pipeline that can call a stored procedure after merges

---

## Setup

1. **Provision the foundation.** Create `TESTING_AGENT_DB`, the schemas `CONFIG`, `TESTS`, `RESULTS`, `DEFECTS`, and `ORCH`, the warehouse `TESTING_AGENT_WH`, and the roles `TESTING_AGENT_ROLE` and `QA_LEAD_ROLE`. The SQL is in `testing-agent`.
2. **Grant read-only access** on each target database to `TESTING_AGENT_ROLE`.
3. **Configure the scope.** Add rows to `CONFIG.MONITORED_SCOPE` for each database and schema to monitor.
4. **Set agent settings.** Add the Cortex model and thresholds to `CONFIG.AGENT_SETTINGS`.
5. **Create the phase objects.** Build the tables, streams, and procedures from each sibling skill:
   - `DETECT_CHANGES`, `REGISTER_CHANGE`
   - `GENERATE_TESTS`
   - `RUN_TEST_SUITE`
   - `REPORT_DEFECTS`, `RECORD_QA_DECISION`, `PROMOTE_TEST`
6. **Create the task graph** and resume it:
   ```sql
   SELECT SYSTEM$TASK_DEPENDENTS_ENABLE('TESTING_AGENT_DB.ORCH.T_00_DETECT_CHANGES');
   ALTER TASK TESTING_AGENT_DB.ORCH.T_90_NIGHTLY_REGRESSION RESUME;
   ```
7. **Attach a resource monitor** to `TESTING_AGENT_WH`.
8. **Upload the semantic model** for Cortex Analyst:
   ```sql
   PUT file://testing_metrics.yaml @TESTING_AGENT_DB.CONFIG.SEMANTIC_MODELS AUTO_COMPRESS = FALSE;
   ```
9. **Register the Cortex Agent tools** listed in `testing-agent`.

---

## Example Configuration

```sql
INSERT INTO TESTING_AGENT_DB.CONFIG.MONITORED_SCOPE (DATABASE_NAME, SCHEMA_NAME, OBJECT_TYPE, CRITICALITY)
VALUES
    ('SALES_DB',   'CURATED', '%',             'HIGH'),
    ('SALES_DB',   'STAGING', 'DYNAMIC_TABLE', 'MEDIUM'),
    ('FINANCE_DB', '%',       '%',             'HIGH');

INSERT INTO TESTING_AGENT_DB.CONFIG.AGENT_SETTINGS (SETTING, MODEL_NAME, VALUE)
VALUES
    ('TESTGEN_MODEL',             '<cortex-model-name>', NULL),
    ('CAUSE_MODEL',               '<cortex-model-name>', NULL),
    ('FALSE_POSITIVE_THRESHOLD',  NULL, '3'),
    ('REGRESSION_TOLERANCE_PCT',  NULL, '0.005');
```

---

## Usage

**Run tests on demand**

```sql
-- Tests for objects changed in the last 24 hours
CALL TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE('CHANGED');

-- Full regression suite (ACTIVE tests only)
CALL TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE('FULL_REGRESSION');

-- A single object
CALL TESTING_AGENT_DB.ORCH.RUN_TEST_SUITE('OBJECT:SALES_DB.CURATED.ORDERS_DT');
```

**Register a change from CI**

```sql
CALL TESTING_AGENT_DB.ORCH.REGISTER_CHANGE(
    'SALES_DB.CURATED.ORDERS_DT', 'DYNAMIC_TABLE', '<commit_hash>', '<pr_id>');
```

**Check recent runs**

```sql
SELECT RUN_ID, RUN_SCOPE, TOTAL_TESTS, PASSED, FAILED, ERRORED, RUN_STATUS, STARTED_AT
FROM TESTING_AGENT_DB.RESULTS.TEST_RUNS
ORDER BY STARTED_AT DESC
LIMIT 10;
```

---

## QA Lead Workflow

1. Review open findings, ordered by severity:
   ```sql
   SELECT * FROM TESTING_AGENT_DB.DEFECTS.V_QA_REVIEW_QUEUE;
   ```
2. Record a decision on each finding:
   ```sql
   CALL TESTING_AGENT_DB.DEFECTS.RECORD_QA_DECISION('<defect_id>', 'ACCEPTED', 'HIGH', 'Confirmed duplicate keys after the merge change');
   CALL TESTING_AGENT_DB.DEFECTS.RECORD_QA_DECISION('<defect_id>', 'REJECTED', NULL, 'Expected behavior for backfilled rows');
   ```
3. Promote useful candidate tests into the regression suite:
   ```sql
   CALL TESTING_AGENT_DB.TESTS.PROMOTE_TEST('<test_id>');
   ```
4. Ask Cortex Analyst questions such as *"What was the pass rate by object last week?"* or *"Which test categories have the highest rejection rate?"*

Each rejection counts as a false positive against the test that produced it. Once a test reaches the configured threshold, it is flagged for the QA Lead to consider retiring.

---

## Object Inventory

| Object | Type | Purpose |
|---|---|---|
| `CONFIG.MONITORED_SCOPE` | Table | Databases, schemas, and object types to watch |
| `CONFIG.AGENT_SETTINGS` | Table | Models, thresholds, tolerances |
| `CONFIG.WATERMARKS` | Table | Last detection point for each change source |
| `TESTS.CHANGE_EVENTS` | Table + Stream | Detected and impacted changes |
| `TESTS.TEST_CASES` | Table | Versioned test catalog |
| `RESULTS.TEST_RUNS` | Table | One header row per run |
| `RESULTS.TEST_RESULTS` | Table | One row per test per run |
| `DEFECTS.DEFECT_LOG` | Table | Deduplicated defects and QA decisions |
| `DEFECTS.V_QA_REVIEW_QUEUE` | Secure view | Open findings for the QA Lead |
| `ORCH.T_00` to `T_30` | Task graph | Detect, generate, run, report |
| `ORCH.T_90_NIGHTLY_REGRESSION` | Task | Nightly full regression run |
| `TESTING_AGENT_WH` | Warehouse | Isolated compute for the agent |

---

## Guardrails

- **Read-only on targets:** destructive fixtures run only on `TESTING_AGENT_SANDBOX_*` clones, which are dropped after each run.
- **Validated SQL:** generated tests must be a single `SELECT` that passes `EXPLAIN` and contains no DML or DDL.
- **Human approval:** generated tests stay `PROPOSED` until the QA Lead promotes them, and defect status changes only through `RECORD_QA_DECISION`.
- **Cost control:** a resource monitor, a warehouse statement timeout, and `QUERY_TAG` tags on every query for cost attribution.
- **Data protection:** PII is masked in sample rows, and samples are shown only through secure views.
- **Traceability:** every defect links to its test, run, and related change event.

---

## Monitoring

```sql
-- Task health
SELECT NAME, STATE, ERROR_MESSAGE, SCHEDULED_TIME, COMPLETED_TIME
FROM TABLE(TESTING_AGENT_DB.INFORMATION_SCHEMA.TASK_HISTORY())
WHERE DATABASE_NAME = 'TESTING_AGENT_DB'
ORDER BY SCHEDULED_TIME DESC
LIMIT 20;

-- Agent credit usage, last 7 days
SELECT DATE_TRUNC('day', START_TIME) AS DAY, SUM(CREDITS_USED) AS CREDITS
FROM SNOWFLAKE.ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY
WHERE WAREHOUSE_NAME = 'TESTING_AGENT_WH'
  AND START_TIME > DATEADD('day', -7, CURRENT_TIMESTAMP())
GROUP BY 1
ORDER BY 1;
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Changes detected hours late | `ACCOUNT_USAGE` latency | Use `INFORMATION_SCHEMA` detection for HIGH-criticality scopes, or register changes from CI |
| Duplicate change events | Watermark not moving forward | Check the `CONFIG.WATERMARKS` merge step in `DETECT_CHANGES` |
| Generated tests discarded | Invalid JSON or failed `EXPLAIN` | Review the Cortex prompt and model, and keep temperature at 0 |
| Many `ERROR` results | Timeouts or missing grants | Check `ERROR_MESSAGE`, grants on the targets, and the warehouse timeout |
| High rejection rate | Over-sensitive templates or thresholds | Tune the boundary percentiles and regression tolerance, then retire flagged tests |
| Leftover sandbox databases | A run failed before cleanup | Run the nightly sweep for `TESTING_AGENT_SANDBOX_*` clones older than 24 hours |

---

## Related Agents

| Agent | Relationship |
|---|---|
| Data Quality Agent (Segment 4) | Monitors data against quality rules, while the Testing Agent tests code and logic changes |
| Reconciliation Agent (Segment 4) | Checks source-to-target accuracy for migrations |
| Code Generation and Pipeline Agents (Segment 3) | Produce the changes the Testing Agent picks up |
| CI/CD and Release Agents (Segment 5) | Use Testing Agent results as evidence for promotion and release |

---

## Ideal Customer Profile

Teams that ship frequent data changes with thin test coverage, where defects regularly reach production.