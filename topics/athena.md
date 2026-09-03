# Amazon Athena — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Amazon Athena is a serverless, interactive query service that runs SQL directly against data in S3 (and other sources via connectors), using the Glue Data Catalog for schema. It's central to **Domain 3: Data Operations & Support**, and appears constantly across ingestion/storage scenarios in Domains 1–2.

| Concept          | Purpose                                                             |
| ----------------- | ------------------------------------------------------------------- |
| Query engine      | Based on Presto/Trino — serverless, pay-per-query                   |
| Data Catalog      | Uses Glue Data Catalog for table/schema metadata                    |
| Workgroup         | Logical grouping for query isolation, cost control, result location |
| CTAS / views      | Create new tables or virtual views from query results                |
| Federated Query   | Query non-S3 sources (RDS, DynamoDB, etc.) via Lambda connectors     |

---

## 2. Core Concepts

- **Serverless, pay-per-query**: Billed by **bytes scanned** (per TB), not by cluster time — no infrastructure to manage.
- **Schema-on-read**: Athena doesn't store data; it applies the Glue Catalog's schema to files in S3 at query time.
- **Supported formats**: CSV, JSON, Parquet, ORC, Avro; **columnar formats (Parquet/ORC) drastically reduce bytes scanned** and thus cost — the single most common Athena cost-optimization exam answer.
- **Partition pruning**: Queries that filter on partition columns (`WHERE year=2026 AND month=09`) only scan matching partitions — requires partitions to be registered in the Catalog (via Crawler, `MSCK REPAIR TABLE`/`ADD PARTITION`, or Partition Projection).

## 3. Performance & Cost Optimization (High-Frequency Exam Topic)

- **Convert to columnar (Parquet/ORC) + compress**: Reduces bytes scanned → lower cost, faster queries.
- **Partition your data** and ensure the Catalog knows about partitions (Crawler or **Partition Projection** — see `glue.md`).
- **Bucket/sort data** within partitions for further pruning on high-cardinality columns.
- **Avoid `SELECT *`**: Only select needed columns — columnar formats let Athena skip unread columns entirely.
- **CTAS (CREATE TABLE AS SELECT)**: Materializes query results as a new table (often in Parquet) — used to pre-aggregate or reformat data for cheaper repeat queries.
- **Compact small files**: Many small files increase per-file overhead and slow query planning, same issue as with Glue/S3 generally.

## 4. Workgroups

- Isolate queries by team/project: separate **query history, result locations (S3), and IAM permissions** per workgroup.
- **Per-query and per-workgroup data usage control**: Set a **bytes-scanned limit** to cap cost/prevent runaway queries — a common exam answer for "prevent analysts from accidentally running an expensive full-table scan."
- **Engine version control**: Workgroups can pin a specific Athena engine version for consistency/testing.

## 5. Federated Query & Athena for Other Sources

- **Federated Query**: Uses **Lambda-based data source connectors** to query data outside S3 — RDS, DynamoDB, DocumentDB, Redshift, on-prem JDBC sources — from a single Athena SQL query, joining across sources.
- Useful when you need a **single query interface across the data lake and operational databases** without first ETL-ing everything into S3.
- **Athena for Apache Spark**: Athena also supports running Spark code (notebooks) directly, serverless, as an alternative to standing up EMR for smaller interactive Spark workloads.

## 6. Views & CTAS

- **Views**: Virtual tables defined by a saved query — no data duplication, always reflects current underlying data.
- **CTAS**: Physically writes new data (as Parquet, etc.) — used for materializing expensive aggregations or converting raw data to an optimized format/layout.
- **INSERT INTO**: Athena supports inserting query results into an existing table (e.g., to incrementally build a CTAS-created table).

## 7. Security

- **IAM policies**: Control who can run queries, on which workgroups/databases/tables.
- **Lake Formation**: Provides fine-grained (row/column/tag-based) access control on top of tables Athena queries — the standard way to restrict specific columns/rows per user/role.
- **Encryption**: Query results in S3 can be encrypted (SSE-S3/SSE-KMS); source data encryption is inherited from S3.

## 8. Common Exam Scenarios & Gotchas

1. **Reduce Athena query cost significantly** → **convert data to Parquet/ORC + partition it** (the default "least effort, most impact" answer).
2. **Prevent runaway/expensive queries from a workgroup** → set a **bytes-scanned limit** on the workgroup.
3. **Query joins data in S3 with data still in RDS/DynamoDB, without ETL-ing everything first** → **Athena Federated Query**.
4. **New partitions aren't showing up in query results** → need to register partitions: run **Crawler**, `MSCK REPAIR TABLE`, or use **Partition Projection**.
5. **Need fine-grained (column/row-level) access control for different analyst teams** → **Lake Formation** permissions on the underlying Catalog tables.
6. **Repeatedly run the same expensive aggregation** → **CTAS** to materialize the result once, query the smaller result table repeatedly.
7. **Isolate query costs/history per team** → separate **Athena Workgroups**.

## 9. Quick Reference Cheat Sheet

- Billing model: **pay per TB scanned**
- Biggest cost lever: **columnar format (Parquet/ORC) + partitioning**
- Partition awareness without a Crawler: **Partition Projection**
- Cap query cost: **workgroup bytes-scanned limit**
- Query non-S3 sources: **Federated Query (Lambda connectors)**
- Materialize expensive query results: **CTAS**
- Fine-grained access control: **Lake Formation**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
