---
name: reconciliation-variance-analysis
description: This skill defines how the Reconciliation Agent classifies record-level and aggregate variances, drills down to the exact mismatched columns, applies tolerance and materiality rules, assigns likely root causes, and traces discrepancies upstream through Snowflake Horizon lineage and access history. Use this skill for any task involving explaining why Snowflake figures do not tie out to a legacy system, diagnosing missing or duplicate records, rounding, truncation, timezone, or type-conversion defects, ranking variances by financial impact, or preparing investigation packs for migration engineers and the QA Lead.
---

# Reconciliation Variance Analysis

## Overview
This skill implements the ANALYZE stage. It turns raw variance rows into explained, prioritized findings so the QA Lead can decide quickly and migration teams can fix root causes early.

## When to Use
- Any run that ends in WARN or FAIL.
- Preparing evidence for a QA Lead INVESTIGATE decision.
- Identifying systemic defects across a migration wave.

## Instructions

1. Drill down to column level for VALUE_MISMATCH rows. Generate one COUNT_IF per compare column from config:

        SELECT
            COUNT_IF(S.INVOICE_AMOUNT IS DISTINCT FROM T.INVOICE_AMOUNT) AS INVOICE_AMOUNT_MISMATCHES,
            COUNT_IF(S.POSTING_DATE   IS DISTINCT FROM T.POSTING_DATE)   AS POSTING_DATE_MISMATCHES,
            COUNT_IF(S.VENDOR_NAME    IS DISTINCT FROM T.VENDOR_NAME)    AS VENDOR_NAME_MISMATCHES
        FROM RAW_DB.ORACLE_EBS.AP_INVOICES S
        JOIN ANALYTICS_DB.FINANCE.AP_INVOICES T
          ON S.INVOICE_ID = T.INVOICE_ID;

    Write the mismatched column names into RECON_VARIANCE.MISMATCHED_COLUMNS for each affected key.

2. Assign root cause hints using deterministic rules before any AI summarization:

    | Signal | ROOT_CAUSE_HINT |
    |---|---|
    | Numeric difference below 1 unit of target scale | ROUNDING_OR_PRECISION |
    | Target string is a prefix of source string | TRUNCATION |
    | Timestamp difference is a whole number of hours | TIMEZONE_CONVERSION |
    | Source value NULL, target value default or empty | NULL_HANDLING |
    | Duplicate keys in target | DUPLICATE_LOAD |
    | MISSING_IN_TARGET clustered after the extract watermark | LATE_ARRIVING_OR_WATERMARK |
    | MISSING_IN_TARGET clustered on one filter value | FILTER_OR_MAPPING_GAP |
    | Character differences only | ENCODING_OR_COLLATION |

3. Check for duplicates, the most common cause of inflated totals:

        SELECT INVOICE_ID, COUNT(*) AS CNT
        FROM ANALYTICS_DB.FINANCE.AP_INVOICES
        GROUP BY ALL
        HAVING COUNT(*) > 1
        ORDER BY CNT DESC
        LIMIT 100;

4. Apply tolerance and materiality. Status rules per check:
    - PASS: difference is zero.
    - WARN: difference is non-zero but within configured tolerance.
    - FAIL: difference exceeds tolerance, or any MISSING_IN_TARGET on a HIGH criticality object.

    Rank FAIL findings by absolute amount impact, then by affected row count, then by object criticality.

5. Trace lineage with Snowflake Horizon to locate where a variance entered:

        SELECT SOURCE_OBJECT_NAME, TARGET_OBJECT_NAME, DISTANCE
        FROM TABLE(SNOWFLAKE.CORE.GET_LINEAGE(
            'ANALYTICS_DB.FINANCE.AP_INVOICES', 'TABLE', 'UPSTREAM', 3));

    Use SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY to identify the query and load that last wrote the affected object, and attach the QUERY_ID to the finding.

6. Build the investigation pack per FAIL finding: variance type, affected keys sample (max 50), mismatched columns, root cause hint, amount impact, lineage path, and responsible pipeline. Store it as VARIANT in RECON_RESULT.DETAIL for the reporting stage.

7. Optional AI summary. After deterministic classification, generate a plain-language explanation for the QA Lead with SNOWFLAKE.CORTEX.COMPLETE, passing only aggregated findings and masked sample keys.

## Coding Conventions
- Deterministic rules decide classification and status; AI output is explanatory only and never changes a status.
- Never copy sensitive field values into variance tables; store keys and column names, and respect masking policies on source objects.
- Limit sample keys to 50 per finding to keep reports readable and storage bounded.
- Express all monetary impacts in the target currency and scale defined in RECON_CONFIG.
- Lineage queries require the role to have Horizon lineage privileges on the traced objects; run them under RECON_AGENT_ROLE.