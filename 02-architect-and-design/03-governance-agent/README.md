# Governance Agent

*Segment 2: Architect & Design*

The Governance Agent builds access controls and data policies into the design stage, so security is set up from the start rather than added afterward. It detects governance triggers, proposes controls, and sends a policy set to the Lead Architect for review.

## Primary Use Case

Embed least-privilege access and compliance controls into every design.

## How It Works

| Stage | Responsibility |
| --- | --- |
| **Senses** | Detects sensitive data, new roles, and design changes that affect access. |
| **Acts** | Proposes role hierarchies, masking and row access policies, and classification tags, and flags designs that violate policy. |
| **Hands off** | Sends the policy set to the Lead Architect, who approves, tightens, or overrides each control. |

## Skills

| Skill | Purpose |
| --- | --- |
| `SKILL` | Coordinates the end-to-end workflow, policy set, and Lead Architect handoff. **Start here.** |
| `governance-sensitive-data-classification` | Identifies sensitive data and proposes classification tags. |
| `governance-role-hierarchy-design` | Designs least-privilege roles, grants, and ownership models. |
| `governance-masking-row-access-policies` | Defines and tests masking and row access policies. |
| `governance-compliance-coverage-reporting` | Validates designs, reports policy coverage, and prepares audit evidence. |

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── governance-sensitive-data-classification.md
        └── governance-role-hierarchy-design.md
        └── governance-masking-row-access-policies.md
        └── governance-compliance-coverage-reporting.md

## Workflow

1. Confirm the policy requirements, classification standards, data owners, and environment boundaries.
2. Detect a governance trigger.
3. Classify the affected data.
4. Design the required roles and grants.
5. Define the masking and row access controls.
6. Validate the design and measure policy coverage.
7. Submit the policy set to the Lead Architect.
8. Release approved controls to the Engineering Team for implementation.

## Human Oversight

The Governance Agent proposes controls but does not have final authority over them. The Lead Architect reviews each control and decides whether to approve, tighten, or override it. No proposed control should be applied to a governed environment until that decision is recorded.

## Snowflake Platform Relevance

| Component | Role |
| --- | --- |
| **Snowflake Horizon** | Provides classification, masking, access policies, and lineage. |
| **Cortex Agents** | Monitor design changes against policy. |
| **Cortex Analyst** | Reports on policy coverage. |

## Who Benefits

- **Security and compliance officers:** See controls tied to data sensitivity.
- **Data owners:** Keep clear authority over who sees what.
- **Auditors:** Review documented, consistent policy decisions.

## Ideal Customer Profile

Regulated organizations in financial services, healthcare, or the public sector, where access-control gaps create audit and breach exposure.