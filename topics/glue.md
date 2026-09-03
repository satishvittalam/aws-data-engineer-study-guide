# AWS Glue — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview
AWS Glue is a serverless data integration service used to discover, prepare, move, and transform data for analytics, ML, and application development. Core components:

| Component | Purpose |
|---|---|
| Glue Data Catalog | Central metadata repository (Hive Metastore-compatible) storing table/schema definitions |
| Glue Crawlers | Scan data stores, infer schema, and populate/update the Data Catalog |
| Glue ETL Jobs | Serverless Spark/Python jobs to transform and move data |
| Glue Studio | Visual job authoring interface (drag-and-drop ETL) |
| Glue DataBrew | No-code visual data preparation/cleaning tool |
| Glue Workflows | Orchestrate multiple crawlers/jobs/triggers as a single unit |
| Glue Schema Registry | Manage and validate schemas for streaming data (Kafka, Kinesis, MSK) |
| Glue Data Quality | Define and evaluate data quality rules (DQDL) on datasets |
| Glue Elastic Views (deprecated) | Combine/replicate data across stores using SQL (largely retired — low exam relevance) |

---

## 2. Glue Data Catalog

- Centralized, persistent metadata store — **databases** contain **tables**; tables hold schema (columns, types), location, partitions, classification.
- Used by Athena, Redshift Spectrum, EMR, Glue ETL jobs as a **Hive Metastore replacement**.
- One Data Catalog **per AWS Region per account** (unless cross-account access is configured).
- **Table properties**: classification (csv, json, parquet, etc.), schema, partition keys, location (S3 path).
- **Partitions**: Improve query performance/cost by pruning scanned data (e.g., Athena only scans relevant partitions).
- Supports **versioning of table schema** (schema history).
- Cross-account access via **resource-based policies** (Glue Catalog resource policy) or **AWS Lake Formation** permissions.

---

## 3. Glue Crawlers

- Connect to a data source (S3, JDBC, DynamoDB), **infer schema**, and create/update tables in the Data Catalog.
- Can detect schema changes and handle them via configurable behavior:
  - **Add new columns**, **Update the table definition**, **Ignore the change**, or mark table as deprecated.
- **Classifiers**: Determine schema/format (built-in for JSON, CSV, Parquet, Avro, ORC, etc.); custom classifiers (grok patterns) can be defined for custom formats.
- Crawlers can be scheduled (cron) or run on-demand.
- Crawling large partitioned datasets can be slow/costly — use **partition indexes** or exclude patterns to optimize.
- Crawler creates one table per "schema" detected; inconsistent file structures within a prefix can cause the crawler to split into multiple tables unexpectedly — a common exam gotcha.

---

## 4. Glue ETL Jobs

### Job Types
- **Spark**: Standard distributed ETL (Python/Scala), most common.
- **Spark Streaming**: Continuous processing of streaming sources (Kinesis, Kafka).
- **Python Shell**: Lightweight, single-node Python scripts (no Spark) — good for simple tasks, orchestration, small data.
- **Ray**: Python jobs using Ray framework for scalable Python workloads (newer).

### Key Concepts
- **DynamicFrame**: Glue's abstraction over Spark DataFrame — designed for semi-structured data, handles schema inconsistencies gracefully (e.g., `ResolveChoice`, `Relationalize`).
- **Job Bookmarks**: Track previously processed data so reruns only process new/changed data (incremental ETL). Must be enabled and requires consistent source (e.g., S3 object timestamps, JDBC primary keys).
- **DPU (Data Processing Unit)**: Unit of compute/memory capacity for a job (1 DPU = 4 vCPU + 16 GB RAM). Billed per DPU-hour (per second, 1-min minimum).
- **Worker Types**: `Standard`, `G.1X`, `G.2X`, `G.025X` (min for low-volume streaming), `G.4X`/`G.8X` (memory-intensive) — choose based on workload.
- **Glue Flex Execution**: Lower-cost, less time-sensitive compute for non-urgent jobs (up to 34% cheaper, longer/variable startup).
- **Connections**: Define JDBC/network connection info (VPC, security groups) reused across jobs/crawlers.
- **Triggers**: Start jobs on schedule, on-demand, or based on event/completion of another job (chaining via Workflows).

### Transformations
- **ResolveChoice**: Resolve ambiguous/conflicting column types.
- **Relationalize**: Flatten nested JSON into multiple relational tables.
- **DropFields / DropNullFields**: Remove unwanted or empty columns.
- **Map / Filter / Join**: Standard row-level transforms.
- **FillMissingValues**: ML-based imputation for missing values.
- Format conversion: read/write CSV, JSON, Parquet, ORC, Avro — Parquet/ORC preferred for analytics (columnar, compressed).

---

## 5. Glue Studio & DataBrew

- **Glue Studio**: Visual, drag-and-drop DAG-based ETL authoring; generates Spark code automatically; supports custom transforms and visual job monitoring.
- **Glue DataBrew**: No-code tool for data analysts to explore, clean, and normalize data using 250+ pre-built transformations ("recipes"); outputs to S3; good for data profiling and quality checks without writing code.

---

## 6. Glue Workflows & Triggers

- **Workflow**: Directed graph orchestrating multiple crawlers, jobs, and triggers as a single logical unit — visualized in the Glue console.
- **Triggers**: 
  - **Schedule-based** (cron)
  - **On-demand**
  - **Event-based** (job completion, EventBridge)
- Use Workflows when you need to coordinate: Crawler → ETL Job → Crawler (update catalog after transform) chains.

---

## 7. Glue Schema Registry

- Central registry to manage and enforce **schemas for streaming data** (Kafka, MSK, Kinesis Data Streams, Kinesis Data Firehose).
- Supports **Avro** and **JSON Schema** formats.
- Enforces **schema compatibility rules** (backward, forward, full) to prevent breaking changes from producers/consumers.
- Reduces data quality issues downstream and enables schema evolution tracking.

---

## 8. Glue Data Quality

- Define data quality rules using **DQDL (Data Quality Definition Language)** — e.g., completeness, uniqueness, value ranges.
- Can run rule sets as part of a Glue ETL job or standalone against Data Catalog tables.
- Produces a **data quality score** and detailed rule pass/fail results — supports pipeline actions on failure (e.g., stop job, quarantine data).
- Integrates with recommendations engine to auto-suggest rules based on data profiling.

---

## 9. Security

- **IAM roles**: Glue jobs/crawlers assume an IAM role for access to S3, other services.
- **Encryption**:
  - At rest: SSE-S3 or SSE-KMS for data in S3; Data Catalog metadata can be encrypted with KMS.
  - In transit: SSL/TLS for JDBC connections.
- **Glue job security configurations**: Attach encryption settings (S3, CloudWatch Logs, job bookmarks) to jobs/crawlers.
- **VPC**: Jobs/crawlers accessing resources inside a VPC (e.g., RDS) need a Glue Connection configured with subnet/security group (requires **ENI** in the VPC — ensure NAT/route for internet-bound calls, e.g., to other AWS APIs).
- **Lake Formation**: Provides fine-grained (row/column/table-level) access control on top of the Glue Data Catalog, centralizing permissions across Athena, Redshift Spectrum, EMR, Glue.

---

## 10. Monitoring & Troubleshooting

- **CloudWatch Logs**: Job run logs, error logs (`/aws-glue/jobs/error` and `/aws-glue/jobs/output`).
- **CloudWatch Metrics**: DPU usage, memory profile, ETL data movement.
- **Glue Job Metrics / Spark UI**: Visualize stage-level performance, identify skew/bottlenecks.
- Common issues:
  - **OOM (Out of Memory) errors** → increase worker type (G.2X) or increase number of workers, or optimize partitioning/skew.
  - **Job running slow due to too many small files ("small file problem")** → use `groupFiles`/`groupSize` options when reading, or compact files upstream.
  - **Crawler creating too many tables from one S3 prefix** → inconsistent schemas/formats in the prefix; enforce consistent structure or use exclude patterns.
  - **Job bookmarks not working** → ensure bookmarks enabled at job creation, source supports it, and transformation context is consistent across runs.

---

## 11. Common Exam Scenarios & Gotchas
1. **Need to catalog data automatically from S3 for Athena queries** → use a **Glue Crawler** to populate the Data Catalog.
2. **Incremental processing of only new files each run** → enable **Job Bookmarks**.
3. **Handling inconsistent/nested JSON schema** → use **DynamicFrame** with `ResolveChoice`/`Relationalize`.
4. **Convert raw CSV/JSON to Parquet for cost-efficient Athena queries** → **Glue ETL job** writing Parquet output.
5. **Analysts need no-code data cleaning** → **Glue DataBrew**.
6. **Need schema validation/compatibility enforcement for streaming producers/consumers** → **Glue Schema Registry**.
7. **Need to chain crawler → job → crawler with dependencies** → **Glue Workflows** with triggers.
8. **Fine-grained access control (column/row level) across Athena/Redshift Spectrum/EMR** → **Lake Formation** on top of Glue Data Catalog.
9. **Low-latency/lightweight script with no need for distributed Spark** → **Python Shell** job type.
10. **Cost optimization for non-time-sensitive batch jobs** → **Glue Flex** execution class.
11. **Automated data quality checks with pass/fail thresholds** → **Glue Data Quality** (DQDL rules).
12. **Job failing with OOM / slow due to data skew** → adjust worker type/count, review partitioning strategy.

---

## 12. Quick Reference Cheat Sheet
- 1 DPU = **4 vCPU + 16 GB RAM**
- Worker types: **Standard, G.1X, G.2X, G.025X, G.4X, G.8X**
- Job Bookmarks → **incremental ETL**
- DynamicFrame → **schema-flexible, semi-structured data**
- Crawler → **schema inference + Data Catalog population**
- Schema Registry formats: **Avro, JSON Schema**
- Data Quality language: **DQDL**
- Fine-grained access control: **Lake Formation**
- No-code prep tool: **DataBrew**
- Visual ETL authoring: **Glue Studio**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
