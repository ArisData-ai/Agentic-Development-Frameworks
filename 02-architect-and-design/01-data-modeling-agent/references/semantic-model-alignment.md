---
name: semantic-model-alignment
description: This skill defines how the Data Modelling Agent aligns physical model objects with governed business terms, using Cortex Analyst semantic definitions to map entities, dimensions, facts, metrics, and synonyms, and producing an alignment report with semantic gaps. Use this skill for any task involving creating or reviewing a semantic view, mapping metrics to model columns, standardizing business terminology, defining synonyms, checking reporting alignment, resolving metric conflicts, or preparing a model for Cortex Analyst consumption.
---

# Semantic Model Alignment

## Overview

This skill keeps the model consistent with business terminology. It confirms that required reporting concepts can be traced to model objects and represented in Cortex Analyst semantic definitions.

## When to Use

Use this skill when:

- A physical draft needs a check against business definitions.
- Metrics, dimensions, or synonyms need mapping.
- Reporting requirements must be traced to model columns.
- Business terms conflict across teams or source systems.

## Instructions

1. Collect approved business terms, metrics, and reporting requirements.
2. Map each term to one model object or one documented expression.
3. Confirm that joins follow the logical model and preserve the metric grain.
4. Add synonyms only when business users or approved documents support them.
5. Record any missing, ambiguous, or conflicting definitions.
6. Include the alignment report in the review package.

Alignment statuses:

    ALIGNED      Term maps cleanly to the model
    AMBIGUOUS    Term has more than one possible mapping
    CONFLICTING  Approved definitions disagree
    MISSING      Model has no object for the term

Semantic view pattern:

    CREATE OR REPLACE SEMANTIC VIEW <DB>.<SEMANTIC_SCHEMA>.SALES_ANALYTICS
        TABLES (
            ORDERS AS <DB>.<MART_SCHEMA>.FCT_ORDER_LINE
                PRIMARY KEY (ORDER_LINE_KEY),
            CUSTOMERS AS <DB>.<MART_SCHEMA>.DIM_CUSTOMER
                PRIMARY KEY (CUSTOMER_KEY)
        )
        RELATIONSHIPS (
            ORDERS_TO_CUSTOMERS AS
                ORDERS (CUSTOMER_KEY) REFERENCES CUSTOMERS
        )
        FACTS (
            ORDERS.NET_AMOUNT AS NET_AMOUNT
        )
        DIMENSIONS (
            CUSTOMERS.CUSTOMER_NAME AS CUSTOMER_NAME
                WITH SYNONYMS = ('<APPROVED_SYNONYM>')
                COMMENT = '<APPROVED_DEFINITION>'
        )
        METRICS (
            ORDERS.TOTAL_NET_SALES AS SUM(ORDERS.NET_AMOUNT)
                COMMENT = '<APPROVED_METRIC_DEFINITION>'
        );

Alignment record:

    term: Net Sales
    status: ALIGNED
    requirement_ref: <REQUIREMENT_ID>
    model_object: FCT_ORDER_LINE.NET_AMOUNT
    semantic_object: ORDERS.TOTAL_NET_SALES
    notes: <REVIEW_NOTES>

## Coding Conventions

- Use approved business terms in semantic names and comments.
- Define each metric once and reuse that definition.
- State the aggregation and grain of each metric.
- Do not expose a relationship that the logical model has not defined.
- Do not invent synonyms, definitions, or metric logic.
- Validate that generated semantic objects can be created before including them for review.