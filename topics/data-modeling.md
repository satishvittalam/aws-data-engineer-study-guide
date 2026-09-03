# Data Modeling — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Data modeling is a conceptual (not service-specific) part of **Domain 2: Data Store Management**. The exam tests whether you understand *how* to structure data for a given store/access pattern — this complements the service-specific knowledge in `databases.md`, `s3-data-lakes.md`, etc.

| Concept                | Purpose                                                                  |
| ------------------------- | ----------------------------------------------------------------------- |
| Normalization            | Reduce redundancy in relational schemas (OLTP-oriented)                 |
| Denormalization           | Duplicate data to reduce joins, favor read performance (OLAP/NoSQL-oriented) |
| Star / Snowflake schema  | Dimensional modeling patterns for data warehouses                       |
| Slowly Changing Dimension | Patterns for handling dimension attribute changes over time             |
| Data Vault               | Modeling approach favoring auditability/history in raw/staging layers   |

---

## 2. Normalization vs. Denormalization

- **Normalization** (1NF/2NF/3NF): Eliminate redundant data by splitting into related tables (e.g., separate `Customers` and `Orders` tables joined by `customer_id`). Favors **write efficiency and data integrity** — standard for OLTP systems (RDS/Aurora).
- **Denormalization**: Intentionally duplicate data to avoid joins at query time (e.g., embedding customer name directly in an `Orders` row). Favors **read performance** — standard for OLAP (Redshift dimension tables), NoSQL (DynamoDB single-table design), and BI-facing datasets.
- Exam framing: *"Frequent joins are slowing down analytical queries"* → **denormalize** (or restructure as a star schema); *"Data integrity/update anomalies in a transactional system"* → **normalize**.

## 3. Dimensional Modeling: Star vs. Snowflake Schema

- **Fact table**: Central table holding measurable, quantitative events (e.g., `sales_fact`: order_id, product_id, customer_id, date_id, amount, quantity) — typically large, append-heavy.
- **Dimension table**: Descriptive attributes about the entities referenced by the fact table (e.g., `product_dim`, `customer_dim`, `date_dim`) — typically smaller, more static.
- **Star schema**: Fact table directly joined to denormalized dimension tables (one join hop) — simpler, fewer joins, faster queries; the default recommended pattern for Redshift/data warehouse modeling.
- **Snowflake schema**: Dimension tables further normalized into sub-dimensions (e.g., `product_dim` → `category_dim`) — reduces redundancy but adds join complexity; generally **not preferred for Redshift-style OLAP** where join minimization matters more than storage savings.
- Exam framing: *"Design a schema for fast BI queries in Redshift"* → **star schema**, using **KEY distribution** on the fact table's join column and **ALL distribution** for small dimension tables (see `databases.md` §5).

## 4. Slowly Changing Dimensions (SCD)

- **Type 1 (Overwrite)**: New value replaces the old value — no history kept. Use when historical accuracy doesn't matter (e.g., correcting a typo).
- **Type 2 (Add new row)**: Insert a new dimension row with a new surrogate key and effective/expiry dates (or a "current flag") — preserves full history, most common pattern for tracking meaningful business changes (e.g., a customer's address history for time-accurate reporting).
- **Type 3 (Add new column)**: Add a "previous value" column alongside the current value — keeps only limited (usually one prior) history; used when only the immediately previous state matters.
- Exam framing: *"Reports must reflect what a customer's region was at the time of each historical order, not their current region"* → **SCD Type 2**.

## 5. Data Vault & Raw/Staging Modeling

- **Data Vault**: Modeling technique for the raw/staging layer of a data warehouse — splits data into **Hubs** (business keys), **Links** (relationships), and **Satellites** (descriptive/historical attributes) — favors auditability, traceability, and resilience to source-schema changes, at the cost of query simplicity (usually transformed into a star schema for the consumption layer).
- Less central to DEA-C01 than star/SCD concepts, but can appear as a distractor/option in schema-design questions about **raw zone** design in a data lake (see `s3-data-lakes.md` §6 zones).

## 6. Modeling for NoSQL (DynamoDB) vs. Relational

- NoSQL modeling is **access-pattern-first**: design the table/index structure around the specific queries the application needs (single-table design, GSIs) rather than normalizing entities — see `databases.md` §4 for DynamoDB-specific patterns (partition/sort key design, hot partition avoidance).
- Contrast with relational/dimensional modeling, which is **entity-first**: model the real-world entities and relationships, then let SQL flexibly query across them.
- Exam framing: *"Design a DynamoDB table"* questions test whether you designed for the **known access patterns**, not whether the schema is "normalized" — a common wrong-answer pattern is over-normalizing a DynamoDB table the way you would an RDS table.

## 7. Common Exam Scenarios & Gotchas

1. **Analytical queries in Redshift are slow due to many joins** → **denormalize into a star schema**.
2. **Need historical accuracy for a changing dimension attribute (e.g., customer address at time of order)** → **SCD Type 2**.
3. **Simple correction of bad data with no need to preserve history** → **SCD Type 1**.
4. **Only need to compare current vs. immediately-previous value** → **SCD Type 3**.
5. **OLTP system experiencing update anomalies / redundant data** → **normalize** the schema.
6. **Designing a DynamoDB table for a known set of application queries** → model around **access patterns** (single-table design, GSIs), not entity normalization.
7. **Raw/staging layer needs to be resilient to upstream schema changes and fully auditable** → consider **Data Vault**-style modeling before transforming into a star schema for consumption.

## 8. Quick Reference Cheat Sheet

- Normalize → **write integrity, OLTP**; Denormalize → **read speed, OLAP/NoSQL**
- Star schema → **fewer joins, preferred for Redshift**; Snowflake → **more normalized dimensions, more joins**
- SCD Type 1 → **overwrite, no history**
- SCD Type 2 → **new row per change, full history** (most common answer)
- SCD Type 3 → **new column, limited (one-step) history**
- DynamoDB modeling → **access-pattern-first**, not entity-normalization-first

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
