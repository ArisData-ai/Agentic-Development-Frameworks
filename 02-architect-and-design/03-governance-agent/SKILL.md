---
name: governance-agent-orchestration
description: This skill defines the end-to-end Governance Agent workflow, which detects sensitive data, new roles, and access-affecting design changes, coordinates classification, role, and policy proposals through Cortex Agents and Snowflake Horizon, and delivers a versioned policy set for Lead Architect review. Use this skill for any task involving Governance Agent setup, security-by-design reviews, least-privilege design, policy set assembly, approval routing, or requests to build, orchestrate, or run governance checks during the Architect & Design segment of a Snowflake program.
---

# Governance Agent Orchestration

## Overview

The Governance Agent builds access controls and data policies into the design stage so security is established from the start. It operates in Segment 2: Architect & Design and follows a Sense, Act, Hand Off model. The agent proposes controls, while the Lead Architect retains final authority over each one.

## When to Use

- Configure or run the Governance Agent for a Snowflake program.
- Respond to newly detected sensitive data, new roles, or access-affecting design changes.
- Assemble a policy set for Lead Architect review.
- Route approved, tightened, or overridden controls to downstream teams.

## Instructions

1. **Establish the governance baseline.** Before evaluating a design, confirm the following inputs:
    - Applicable regulatory and internal policy requirements
    - Data sensitivity levels and classification tags
    - Data owners for each domain
    - Environment boundaries and naming standards

2. **Sense governance triggers.** Monitor for these events:
    - Sensitive data in new or changed objects
    - New or modified roles
    - Design changes that affect access paths, joins, sharing, or data movement

3. **Run the proposal workflow.** Process each trigger in this order:
    - Classify data with `governance-sensitive-data-classification`.
    - Design roles with `governance-role-hierarchy-design`.
    - Define controls with `governance-masking-row-access-policies`.
    - Validate the design with `governance-compliance-coverage-reporting`.

4. **Assemble the policy set.** Link each proposed control to its trigger, affected objects, data sensitivity, policy requirement, and rationale.

5. **Hand off to the Lead Architect.** Submit the policy set for a decision on each control:
    - **Approve:** Accept the control as proposed.
    - **Tighten:** Strengthen the control and record the change.
    - **Override:** Replace the control and record the reason.

6. **Enforce the approval gate.** Do not apply a proposed control to any governed environment without a recorded Lead Architect decision. Keep unresolved violations open and visible.

7. **Release approved outputs.** Pass only approved controls to the Engineering Team as implementation-ready artifacts. Retain decisions, reviewers, and timestamps for audit.

## Coding Conventions

- Assign a unique identifier and version to every policy set and control.
- Keep proposals idempotent and safe to rerun.
- Separate proposed, approved, and deployed states.
- Store no credentials, secrets, or sensitive sample values in agent outputs.
- Use this policy set structure:

        policy_set:
          id: GOV-PS-001
          version: 1
          status: PENDING_REVIEW
          trigger: NEW_SENSITIVE_DATA
          reviewer: LEAD_ARCHITECT
          controls:
            - control_id: GOV-CTRL-001
              control_type: MASKING_POLICY
              target: ANALYTICS_DB.CUSTOMER.DIM_CUSTOMER.EMAIL
              sensitivity: CONFIDENTIAL
              requirement: PII_PROTECTION_STANDARD
              rationale: Restrict direct identifiers to approved roles
              decision: PENDING
              decision_notes: null

## References

| Skill file | Coverage |
| --- | --- |
| `references/governance-sensitive-data-classification.md` | Sensitive data detection, classification review, and tagging proposals |
| `references/governance-role-hierarchy-design.md` | Least-privilege role hierarchies, access roles, and grant design |
| `references/governance-masking-row-access-policies.md` | Masking policies, row access policies, entitlement mappings, and policy testing |
| `references/governance-compliance-coverage-reporting.md` | Design violation checks, policy coverage metrics, and audit-ready reporting |