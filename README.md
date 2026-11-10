# Agentic-Development-Frameworks

A collection of agent skills covering the full data project lifecycle, from discovery through to production operations. The lifecycle is split into 6 segments, and each segment has 3 specialised agents.

## Lifecycle Overview

| # | Segment | Agents |
|---|---------|--------|
| 01 | **Discover & Assess** | Requirements Agent · Discovery Agent · Data Assessment Agent |
| 02 | **Architect & Design** | Data Modelling Agent · Architecture Agent · Governance Agent |
| 03 | **Develop & Integrate** | Pipeline Agent · Code Generation Agent · Documentation Agent |
| 04 | **Test & Validate** | Data Quality Agent · Testing Agent · Reconciliation Agent |
| 05 | **Deploy & Release** | CI/CD Agent · Release Agent · Deployment Validation Agent |
| 06 | **Operate & Optimize** | Cost Optimization Agent · Observability Agent · Remediation Agent |

## Repository Structure

    Agentic-Development-Frameworks/
    ├── 01-discover-and-assess/
    │ ├── 01-requirements-agent/
    │ ├── 02-discovery-agent/
    │ └── 03-data-assessment-agent/
    ├── 02-architect-and-design/
    │ ├── 01-data-modelling-agent/
    │ ├── 02-architecture-agent/
    │ └── 03-governance-agent/
    ├── 03-develop-and-integrate/
    │ ├── 01-pipeline-agent/
    │ ├── 02-code-generation-agent/
    │ └── 03-documentation-agent/
    ├── 04-test-and-validate/
    │ ├── 01-data-quality-agent/
    │ ├── 02-testing-agent/
    │ └── 03-reconciliation-agent/
    ├── 05-deploy-and-release/
    │ ├── 01-ci-cd-agent/
    │ ├── 02-release-agent/
    │ └── 03-deployment-validation-agent/
    ├── 06-operate-and-optimize/
    │ ├── 01-cost-optimization-agent/
    │ ├── 02-observability-agent/
    │ └── 03-remediation-agent/
    └── README.md


### Agent Folder Layout

Every agent folder follows the same layout:
NN-<agent-name>/
├── SKILL.md # Entry point: the agent’s role, workflow, and when to load each reference
├── README.md # Human-readable overview of the agent
└── references/ # Task-specific instructions, loaded by SKILL.md only when needed
├── <agent>-<task-1>.md
├── <agent>-<task-2>.md
└── ...


- **`SKILL.md`** is the single entry point. The agent always starts here.
- **`references/`** holds detailed, task-specific files. `SKILL.md` points to them, and they are loaded only when that task comes up, which keeps the agent's context small.
- **`README.md`** explains the agent for people browsing the repo.

**Example (Requirements Agent):**

01-requirements-agent/
├── SKILL.md
├── README.md
└── references/
├── requirements-extraction-...md
├── requirements-governance-...md
├── requirements-intake-...md
└── requirements-validation-...md


## Segments

### 01 · Discover & Assess
Gathers business requirements, finds the source systems, and checks the current state of the data before any design work starts.
- **Requirements Agent**: Captures and structures business and technical requirements.
- **Discovery Agent**: Inventories source systems, objects, and dependencies.
- **Data Assessment Agent**: Profiles source data for volume, quality, and complexity.

### 02 · Architect & Design
Defines the target data model, platform architecture, and governance controls.
- **Data Modelling Agent**: Designs logical and physical data models.
- **Architecture Agent**: Lays out the target-state architecture and integration patterns.
- **Governance Agent**: Defines access control, data classification, and policies.

### 03 · Develop & Integrate
Builds the pipelines and code, and documents them.
- **Pipeline Agent**: Builds ingestion and transformation pipelines.
- **Code Generation Agent**: Generates SQL and application code from designs.
- **Documentation Agent**: Produces technical documentation for what was built.

### 04 · Test & Validate
Checks that the data and code are correct before release.
- **Data Quality Agent**: Defines and runs data quality checks.
- **Testing Agent**: Creates and runs unit and integration tests.
- **Reconciliation Agent**: Reconciles source and target data for completeness and accuracy.

### 05 · Deploy & Release
Moves validated changes into production safely.
- **CI/CD Agent**: Sets up and manages build and deployment pipelines.
- **Release Agent**: Coordinates release planning, versioning, and release notes.
- **Deployment Validation Agent**: Verifies deployments after release.

### 06 · Operate & Optimize
Keeps production running efficiently and reliably.
- **Cost Optimization Agent**: Finds and recommends compute and storage savings.
- **Observability Agent**: Monitors pipeline health, performance, and failures.
- **Remediation Agent**: Diagnoses issues and applies or proposes fixes.