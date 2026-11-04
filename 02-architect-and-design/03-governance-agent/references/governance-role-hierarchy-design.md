---
name: governance-role-hierarchy-design
description: This skill defines the least-privilege role design process, which converts personas, data domains, and sensitivity classifications into Snowflake functional roles, access roles, database roles, grants, and ownership models for architectural review. Use this skill for any task involving RBAC design, role hierarchy creation, access role mapping, future grants, managed access schemas, privilege reviews, separation of duties, or requests to design, tighten, or audit who can access Snowflake objects.
---

# Governance Role Hierarchy Design

## Overview

This skill creates a role model that gives each persona only the access it requires. It separates business responsibilities from object privileges so access remains clear, reusable, and auditable.

## When to Use

- Design RBAC for a new Snowflake platform or data domain.
- Add a persona, application, or service role.
- Review privilege creep or excessive grants.
- Align access with approved data sensitivity classifications.

## Instructions

1. **Map access requirements.** Identify each persona, its data domains, required actions, environment, and permitted sensitivity levels.

2. **Separate role layers.** Use functional roles for job responsibilities and access roles for object privileges:

        USER
          -> FUNCTIONAL ROLE
              -> DATABASE ROLE OR ACCESS ROLE
                  -> OBJECT PRIVILEGES

3. **Create scoped access roles.** Group privileges by database, schema, and access level:

        USE ROLE SECURITYADMIN;

        CREATE ROLE IF NOT EXISTS FR_CUSTOMER_ANALYST;

        USE DATABASE ANALYTICS_DB;
        CREATE DATABASE ROLE IF NOT EXISTS DBR_CUSTOMER_READ;

        GRANT USAGE ON SCHEMA ANALYTICS_DB.CUSTOMER
            TO DATABASE ROLE DBR_CUSTOMER_READ;
        GRANT SELECT ON ALL TABLES IN SCHEMA ANALYTICS_DB.CUSTOMER
            TO DATABASE ROLE DBR_CUSTOMER_READ;
        GRANT SELECT ON FUTURE TABLES IN SCHEMA ANALYTICS_DB.CUSTOMER
            TO DATABASE ROLE DBR_CUSTOMER_READ;

        GRANT DATABASE ROLE DBR_CUSTOMER_READ
            TO ROLE FR_CUSTOMER_ANALYST;

4. **Connect the role hierarchy.** Grant custom functional roles to the approved administrative role so ownership and management paths remain clear:

        GRANT ROLE FR_CUSTOMER_ANALYST TO ROLE SYSADMIN;

5. **Protect sensitive schemas.** Use managed access schemas where centralized grant control is required:

        CREATE SCHEMA IF NOT EXISTS ANALYTICS_DB.RESTRICTED
            WITH MANAGED ACCESS;

6. **Enforce separation of duties.** Keep security administration, policy administration, object ownership, data engineering, and audit access in distinct roles where the operating model allows it.

7. **Flag privilege risks.** Report broad grants, direct user privileges, administrative role use, cross-environment access, and service roles with unnecessary write access.

8. **Prepare the role proposal.** Include the hierarchy, role purpose, grants, justified exceptions, and affected objects for Lead Architect review.

## Coding Conventions

- Use consistent role prefixes such as `FR_`, `AR_`, `DBR_`, and `SR_`.
- Grant privileges to roles rather than directly to users.
- Reserve `ACCOUNTADMIN` for controlled administrative tasks.
- Do not grant `ALL PRIVILEGES` without a documented exception.
- Keep development, test, and production access separate.
- Write grant scripts so they can be rerun safely.