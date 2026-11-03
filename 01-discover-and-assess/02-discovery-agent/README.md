# Discovery Agent

**Segment 1: Discover & Assess**

The Discovery Agent continuously maps the legacy systems, sources, and dependencies that a data initiative must account for. It replaces manual system inventories at the start of migration and modernization programs with a verified, reviewable view of the source estate.

## What It Does

| Phase | Responsibility |
| --- | --- |
| **Sense** | Detects new or changed systems, schemas, jobs, and interfaces across the source estate. |
| **Act** | Catalogs assets, traces upstream and downstream dependencies, and flags undocumented or orphaned components. |
| **Hand Off** | Presents the inventory and dependency map to the Lead Validator, who confirms scope and resolves ambiguous or disputed findings. |

## Skill Files

| Skill | Purpose |
| --- | --- |
| `SKILL` | Coordinates sweeps, phase execution, state tracking, and exception handling. |
| `discovery-source-metadata-extraction` | Connects to source systems, captures metadata snapshots, and detects changes. |
| `discovery-asset-cataloging` | Standardizes assets and stores governed catalog records. |
| `discovery-dependency-mapping` | Traces dependencies and flags undocumented or orphaned components. |
| `discovery-validation-handoff` | Packages findings, records Lead Validator decisions, and releases confirmed scope. |

Start with `SKILL.md`, which references the other skills.

## Repository Structure

    discovery-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
       └── discovery-source-metadata-extraction.md
       └── discovery-asset-cataloging.md
       └── discovery-validation-handoff.md
       └── discovery-dependency-mapping.md


## Workflow

1. Register the in-scope source systems.
2. Start a discovery sweep.
3. Extract metadata and compare it with the latest complete sweep.
4. Add new and changed assets to the catalog.
5. Trace dependencies and flag risks.
6. Send the review package to the Lead Validator.
7. Record each decision and rationale.
8. Release confirmed scope to the Data Assessment Agent.

## Snowflake Platform Components

| Component | Role |
| --- | --- |
| **Openflow** | Connects to source systems to pull metadata for cataloging. |
| **Cortex Agents** | Plan and run recurring discovery sweeps. |
| **Snowflake Horizon** | Stores catalog and lineage metadata with governance. |

## Outputs

- Asset inventory
- Upstream and downstream dependency map
- Change summary for each sweep
- List of undocumented, orphaned, and unresolved components
- Lead Validator review package
- Decision log
- Confirmed scope for the Data Assessment Agent

## Human Oversight

The agent identifies findings and provides supporting evidence, but it does not make the final scope decision. The Lead Validator confirms scope and resolves ambiguous or disputed findings. Only confirmed assets move to the Data Assessment Agent.

## Who Benefits

- **Migration leads:** Start with a verified view of what exists and what depends on it.
- **Enterprise architects:** Spot hidden coupling before it becomes a delivery risk.
- **IT operations:** Retire redundant systems with evidence.

## Ideal Customer Profile

Organizations with aging, poorly documented data estates planning a cloud migration, where no one has a complete picture of systems and dependencies.