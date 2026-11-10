# Cost Optimization Agent

*Segment 6: Operate & Optimize*

The Cost Optimization Agent watches Snowflake credit consumption and finds savings without hurting performance. It spots usage spikes, idle warehouses, and inefficient queries. It then estimates what each change would save and presents ranked recommendations to the Operations Lead, who approves, adjusts, or declines each one. The agent never applies a change unless that approval is recorded.

---

## Why It Exists

Fast-growing Snowflake accounts often see rising, unpredictable credit bills with little visibility into who drives them. This agent applies FinOps discipline to credit usage on a continuous basis:

| Audience | Value |
|---|---|
| FinOps and finance teams | Visibility into spend drivers by warehouse, user, role, and team |
| Platform owners | Cost control without throttling workloads |
| Budget holders | Early warning before overruns |

---

## How It Works

    SENSE                         ACT                              HAND OFF
    ─────────────────────         ─────────────────────────        ──────────────────────────
    Usage spikes          ──┐     Quantify waste                   Ranked review queue
    Idle warehouses       ──┼──>  Model resize / suspend impact ─> Operations Lead decides
    Inefficient queries   ──┘     Draft recommendations            Approved changes applied
                                                                   Every decision audited

| Stage | Output | Implemented in |
|---|---|---|
| Sense | `COST_SIGNALS` | Anomaly, warehouse, and query analysis procedures |
| Act | `COST_RECOMMENDATIONS` (status `PENDING`) | Snowpark modeling procedure |
| Hand off | `COST_ACTION_LOG`, applied `ALTER WAREHOUSE` | Approval procedure |

---

## Snowflake Components

| Component | Role in the agent |
|---|---|
| `SNOWFLAKE.ACCOUNT_USAGE` | Source metadata: metering, load, events, query history, query attribution |
| Snowpark (Python) | Savings and performance-risk modeling |
| `SNOWFLAKE.ML.ANOMALY_DETECTION` | Seasonal-aware credit anomaly detection |
| Tasks | Daily scan, model, and rank pipeline |
| Resource Monitors and Budgets | Hard controls and overrun alerts |
| Cortex Analyst (semantic view) | Natural language questions on credit trends and cost drivers |
| Cortex Agents | Orchestrates monitoring, the review queue, and conversational access |

---

## Folder Structure

    cost-optimization-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── cost-usage-anomaly-detection.md
        └── cost-warehouse-efficiency-analysis.md
        └── cost-query-efficiency-analysis.md
        └── cost-savings-impact-modeling.md
        └── cost-recommendation-handoff.md
        └── cost-analyst-and-agent-interface.md

---

## Prerequisites

- A role that can create databases, warehouses, roles, tasks, and agents (for example `SYSADMIN` together with `SECURITYADMIN`)
- Access to the `SNOWFLAKE` database through the `USAGE_VIEWER` and `GOVERNANCE_VIEWER` database roles
- Cortex Analyst and Cortex Agents available in the account's region
- An email notification integration (`FINOPS_EMAIL_INT`) for alerts
- A role that owns the approval procedure and holds `MODIFY` on the target warehouses

---

## Quick Start

1. **Provision the foundation.** Create `COST_OPS_DB.FINOPS`, `FINOPS_WH`, `FINOPS_AGENT_ROLE`, `FINOPS_APPROVER_ROLE`, and the core tables, following the `cost-optimization-agent` skill.
2. **Deploy detection.** Create `SP_DETECT_USAGE_SPIKES`, `SP_DETECT_IDLE_WAREHOUSES`, and `SP_DETECT_INEFFICIENT_QUERIES`.
3. **Deploy modeling.** Create `SP_MODEL_AND_RANK_RECOMMENDATIONS`.
4. **Deploy the hand-off.** Create `V_REVIEW_QUEUE` and `SP_DECIDE_RECOMMENDATION`, then grant them to `FINOPS_APPROVER_ROLE`.
5. **Schedule.** Create and resume the task graph rooted at `T_COST_SCAN_ROOT`.
6. **Enable conversation.** Create `SV_CREDIT_COST` and `COST_OPTIMIZATION_AGENT`.
7. **Validate.** Run one scan by hand and check the outputs:

        EXECUTE TASK COST_OPS_DB.FINOPS.T_COST_SCAN_ROOT;
        SELECT * FROM COST_OPS_DB.FINOPS.COST_SIGNALS ORDER BY DETECTED_AT DESC;
        SELECT * FROM COST_OPS_DB.FINOPS.V_REVIEW_QUEUE ORDER BY PRIORITY_RANK;

---

## Reviewing Recommendations

    -- Approve as proposed
    CALL COST_OPS_DB.FINOPS.SP_DECIDE_RECOMMENDATION('<REC_ID>', 'APPROVED', NULL, 'Approved in weekly review');

    -- Approve with adjustment
    CALL COST_OPS_DB.FINOPS.SP_DECIDE_RECOMMENDATION('<REC_ID>', 'ADJUSTED',
        'ALTER WAREHOUSE ANALYTICS_WH SET AUTO_SUSPEND = 120', 'Keep cache warm for BI');

    -- Decline
    CALL COST_OPS_DB.FINOPS.SP_DECIDE_RECOMMENDATION('<REC_ID>', 'DECLINED', NULL, 'Month-end close workload');

---

## Example Questions for the Agent

- Which warehouses drove the most credits last week?
- Why did credits rise compared with the previous 7 days?
- Who are the top 10 users by attributed credits this month?
- What savings actions are pending, and how much would they save?
- Which teams are on track to exceed their monthly budget?

---

## Guardrails

- **Human in the loop.** No configuration change runs without an Operations Lead decision.
- **Least privilege.** The agent role reads usage metadata and writes only to `COST_OPS_DB.FINOPS`.
- **Allow-listed actions.** Only `ALTER WAREHOUSE ... SET` statements can be applied automatically. Query rewrites go to the owning team.
- **Full audit.** Every decision is logged with the approver identity and the final SQL.
- **Rollback ready.** Each recommendation stores its prior configuration, so the original state can be restored.
- **Data latency.** `ACCOUNT_USAGE` can lag by up to about 3 hours, so the most recent 3 hours are excluded from analysis.

---

## Configuration

| Parameter | Default | Purpose |
|---|---|---|
| Spike z-score threshold | 3.0 | Sensitivity of spike detection |
| Spike uplift threshold | 25% | Minimum increase over baseline |
| Baseline window | 28 days | Rolling comparison period |
| Idle ratio threshold | 30% | Flags warehouses for suspend or resize review |
| Recommended auto-suspend | 60 s | Default target for ad hoc and BI warehouses |
| Analysis window | 14 days | Lookback for warehouse and query analysis |
| Credit price | Account-specific | Converts credits to currency in reports |

For production, store these values in a `COST_CONFIG` table rather than hard-coding them.

---

## Ideal Customer Profile

Fast-growing Snowflake customers facing rising, unpredictable credit bills and limited visibility into who drives them.