# Code Generation Agent

**Segment 3: Develop & Integrate**

The Code Generation Agent produces SQL and transformation code from approved designs and follows team standards. It generates Snowflake-compatible code, converts legacy logic, and flags statements it cannot translate with confidence. The Engineering Team reviews, edits, and approves all code before it is committed.

## Primary Use Case

The agent removes repetitive hand-coding and speeds legacy code conversion.

## How It Works

1. **Senses:** Detects approved mappings and transformation specifications.
2. **Acts:** Generates SQL, converts legacy logic to Snowflake-compatible code, and flags statements it cannot translate with confidence.
3. **Hands off:** Presents the code to the Engineering Team, who review, edit, and approve it before commit.

## Skills

| Skill | Purpose |
| --- | --- |
| `SKILL` | Defines the primary workflow, control boundaries, and output expectations. |
| `sql-transformation-generation` | Generates Snowflake SQL and Snowpark code from approved designs. |
| `legacy-code-conversion` | Converts legacy SQL and stored procedures and assigns confidence ratings. |
| `coding-standards-retrieval` | Retrieves coding standards and prior patterns through Cortex Search. |
| `engineering-review-handoff` | Packages code for review and manages approvals and revisions. |

## Folder Structure

    data-modelling-agent/
    ├── README.md
    ├── SKILL.md
    ├── references/
        └── sql-transformation-generation.md
        └── legacy-code-conversion.md
        └── coding-standards-retrieval.md
        └── engineering-review-handoff.md

## Snowflake Platform Components

| Component | Role |
| --- | --- |
| Cortex Agents | Drives code generation and revision. |
| Cortex Search | Retrieves coding standards and prior patterns. |
| Snowpark | Runs generated transformation logic. |

## Inputs

- Approved source-to-target mappings
- Approved transformation specifications
- Legacy SQL and stored procedures approved for conversion
- Coding standards and prior patterns

## Outputs

- Draft Snowflake SQL and Snowpark code
- Traceability to the source specification or legacy object
- Confidence ratings and code flags
- Review manifest and pull request summary

## Human Approval

The Engineering Team retains final authority over all generated code.

- The agent does not commit, merge, deploy, or promote code.
- The agent does not generate code from unapproved specifications.
- The agent does not invent mappings or business rules.
- Unresolved `BLOCKER` flags prevent code from being marked ready for commit.

## Getting Started

1. Add the skill folders to your agent environment.
2. Create a Cortex Search service for approved coding standards and prior patterns.
3. Grant the agent role access to approved specifications, standards, and required Snowflake objects.
4. Submit an approved mapping, transformation specification, or legacy object.
5. Review the generated code and manifest.
6. Approve the code or request changes, then commit approved code through your standard source control process.

## Who Benefits

- **Developers:** Start from consistent, standards-aligned code.
- **Migration teams:** Convert large legacy code bases faster.
- **Technical leads:** Review against a uniform baseline.

## Ideal Customer Profile

Enterprises migrating large volumes of legacy SQL or stored procedures, where manual conversion is slow and error-prone.