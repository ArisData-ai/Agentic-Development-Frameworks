---
name: pipeline-design-intake
description: This skill defines the intake process that evaluates approved designs and source mappings, confirms build readiness, and produces a validated pipeline build manifest and mapping gap log for Snowflake ELT work. Use this skill for any task involving checking whether a design is ready to build, reviewing source-to-target mappings, parsing mapping spreadsheets, finding missing fields or transformation rules, resolving SAP, Oracle, or Workday mapping questions, or creating a build manifest.
---

# Pipeline Design Intake

## Overview
This skill controls the Sense stage of the Pipeline Agent. It confirms that a design and its source mappings are approved and buildable, then converts them into a build manifest. It also records gaps that require human resolution.

## When to Use
- A design or source mapping has been approved for build.
- An approved mapping has changed.
- A team needs to know whether a pipeline can enter build.
- A mapping contains missing, ambiguous, or conflicting information.

## Instructions
1. **Confirm approval.** Record the design version, mapping version, approver, and approval date. Stop if approval evidence is missing.
2. **Check the required inputs.** Confirm that the package identifies:
    - Source system, objects, and fields
    - Target database, schema, objects, and fields
    - Data types and keys
    - Load pattern and required data freshness
    - Transformation rules and dependencies
    - Data sensitivity and applicable access controls
3. **Validate the mapping.** Check for unmapped required fields, incompatible data types, undefined rules, duplicate target columns, and unknown dependencies.
4. **Classify each gap.**
    - **Blocking:** The agent cannot build a correct pipeline.
    - **Non-blocking:** The affected work can be isolated while other work continues.
5. **Create the manifest.** Include only approved, buildable scope. Record gaps separately.
6. **Set build status.** Use `READY_FOR_BUILD` when no blocking gaps remain. Otherwise, use `BLOCKED_MAPPING_GAP` and send the gap log to the Engineering Team.

Never resolve a mapping gap by assumption. Unresolved items stay flagged until a named owner provides an answer.

## Coding Conventions
- Use one manifest per pipeline and keep it in version control.
- Reference approved artifacts instead of copying their full contents.
- Assign each gap a unique ID, owner, severity, and status.
- Use this manifest structure:

        pipeline:
          name: sales_orders_elt
          design_version: "2.1"
          mapping_version: "1.4"
          approved_by: "Lead Architect"
        source:
          system: "SAP"
          objects: ["SALES_ORDERS"]
        target:
          database: "${TARGET_DATABASE}"
          schema: "CURATED"
          objects: ["SALES_ORDERS"]
        load:
          pattern: "incremental"
          target_lag: "1 hour"
        gaps:
          - id: "GAP-001"
            severity: "blocking"
            object: "SALES_ORDERS.NET_VALUE"
            issue: "Currency conversion rule is undefined"
            owner: "Engineering Team"
            status: "open"