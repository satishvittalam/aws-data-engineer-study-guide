# AWS Glue — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

AWS Glue is a serverless data integration service used to discover, prepare, move, and transform data for analytics, ML, and application development. Core components:

| Component                       | Purpose                                                                                  |
| -------------------------------- | ------------------------------------------------------------------------------------------ |
| Glue Data Catalog               | Central metadata repository (Hive Metastore-compatible) storing table/schema definitions |
| Glue Crawlers                   | Scan data stores, infer schema, and populate/update the Data Catalog                     |
| Glue ETL Jobs                   | Serverless Spark/Python jobs to transform and move data                                  |
| Glue Studio                     | Visual job authoring interface (drag-and-drop ETL)                                       |
| Glue Interactive Sessions       | Serverless, on-demand Jupyter notebook backend for iterative Spark/Python development     |
| Glue DataBrew                   | No-code visual data preparation/cleaning tool                                            |
| Glue Workflows                  | Orchestrate multiple crawlers/jobs/triggers as a single unit                             |
| Glue Schema Registry            | Manage and validate schemas for streaming data (Kafka, Kinesis, MSK)                     |
| Glue Data Quality               | Define and evaluate data quality rules (DQDL) on datasets                                |
| Glue ML Transforms              | Built-in machine learning transforms (e.g., FindMatches) usable inside ETL jobs           |
| Glue Elastic Views (deprecated) | Combine/replicate data across stores using SQL (largely retired — low exam relevance)    |

---

## 2. Glue Data Catalog

- Centralized, persistent metadata store — **databases** contain **tables**; tables hold schema (columns, types), location, partitions, classification.
- Used by Athena, Redshift Spectrum, EMR, Glue ETL jobs as a **Hive Metastore replacement**.
- One Data Catalog **per AWS Region per account** (unless cross-account access is configured).
- **Table properties**: classification (csv, json, parquet, etc.), schema, partition keys, location (S3 path).
- **Partitions**: Improve query performance/cost by pruning scanned data (e.g., Athena only scans relevant partitions).
- Supports **versioning of table schema** (schema history).
- Cross-account access via **resource-based policies** (Glue Catalog resource policy) or **AWS Lake Formation** permissions.

### Partition Projection (Athena, catalog-adjacent)

- When S3 data follows a **predictable partition scheme** (e.g., `year=2026/month=09/day=03`), Athena's **Partition Projection** can compute partitions on the fly from table properties instead of relying on the Data Catalog having each partition registered.
- Avoids running a **Glue Crawler** just to discover new partitions — reduces crawler cost/latency and eliminates "stale partition" lag between new data landing and it being queryable.
- Common exam answer for: *"new date-partitioned data lands hourly in S3 and must be queryable in Athena immediately, with minimal operational overhead."*
- Trade-off: partition scheme must be known/predictable in advance (range, enum, or date-based) — not suited for arbitrary/discovered partition values, where a Crawler is still the right tool.

---

## 3. Glue Crawlers

- Connect to a data source (S3, JDBC, DynamoDB), **infer schema**, and create/update tables in the Data Catalog.
- Can detect schema changes and handle them via configurable behavior:
  * **Add new columns**, **Update the table definition**, **Ignore the change**, or mark table as deprecated.
- **Classifiers**: Determine schema/format (built-in for JSON, CSV, Parquet, Avro, ORC, etc.); custom classifiers (grok patterns) can be defined for custom formats.
- Crawlers can be scheduled (cron) or run on-demand.
- Crawling large partitioned datasets can be slow/costly — use **partition indexes** or exclude patterns to optimize (or replace with **Partition Projection**, see §2, when the partition layout is predictable).
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
- **Glue Auto Scaling**: For Spark jobs, Glue can automatically scale the number of workers **up or down during execution** (between a configured minimum and the job's max capacity) based on Spark task parallelism at each stage. Reduces cost for jobs with uneven workload across stages, without requiring manual DPU tuning — common exam answer for *"reduce Glue job cost when resource needs vary significantly during a single run."*
- **Glue Flex Execution**: Lower-cost, less time-sensitive compute for non-urgent jobs (up to 34% cheaper, longer/variable startup).
- **Job Timeout & Retries**: Each job has a configurable timeout (max runtime before Glue stops it) and a number of automatic retries on failure — important for controlling runaway costs and building resilient scheduled pipelines.
- **Connections**: Define JDBC/network connection info (VPC, security groups) reused across jobs/crawlers.
- **Triggers**: Start jobs on schedule, on-demand, or based on event/completion of another job (chaining via Workflows).

### Transformations

- **ResolveChoice**: Resolve ambiguous/conflicting column types.
- **Relationalize**: Flatten nested JSON into multiple relational tables.
- **DropFields / DropNullFields**: Remove unwanted or empty columns.
- **Map / Filter / Join**: Standard row-level transforms.
- **FillMissingValues**: ML-based imputation for missing values.
- Format conversion: read/write CSV, JSON, Parquet, ORC, Avro — Parquet/ORC preferred for analytics (columnar, compressed).

### Glue ML Transforms

- **FindMatches**: Built-in ML transform that identifies duplicate or matching records **without requiring rule-based/exact-match logic** — trained on labeled examples you provide, then applied at scale in an ETL job. Common exam answer for *"deduplicate customer records that don't match exactly (e.g., typos, formatting differences)"* — distinct from a simple `DropDuplicates` transform, which only catches exact matches.
- Used within a Glue ETL job like any other transform, but requires an initial labeling/training step via the Glue console.

---

## 5. Glue Studio, Interactive Sessions & DataBrew

- **Glue Studio**: Visual, drag-and-drop DAG-based ETL authoring; generates Spark code automatically; supports custom transforms and visual job monitoring.
- **Glue Interactive Sessions**: Serverless, on-demand Spark backend for **Jupyter notebooks** (via Glue Studio notebooks, SageMaker, or local IDE) — lets developers iteratively write and test ETL/Spark code without waiting for a full job to start/stop each time. Billed per session by DPU-second; sessions auto-terminate after a configurable idle timeout. Common exam answer for *"data engineers want to interactively develop and test Glue Spark scripts before deploying them as a job, minimizing cost during development."*
- **Glue DataBrew**: No-code tool for data analysts to explore, clean, and normalize data using 250+ pre-built transformations ("recipes"); outputs to S3; good for data profiling and quality checks without writing code.

---

## 6. Glue Workflows & Triggers

- **Workflow**: Directed graph orchestrating multiple crawlers, jobs, and triggers as a single logical unit — visualized in the Glue console.
- **Triggers**:
  * **Schedule-based** (cron)
  * **On-demand**
  * **Event-based** (job completion, EventBridge)
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
  * At rest: SSE-S3 or SSE-KMS for data in S3; Data Catalog metadata can be encrypted with KMS.
  * In transit: SSL/TLS for JDBC connections.
- **Glue job security configurations**: Attach encryption settings (S3, CloudWatch Logs, job bookmarks) to jobs/crawlers.
- **VPC**: Jobs/crawlers accessing resources inside a VPC (e.g., RDS) need a Glue Connection configured with subnet/security group (requires **ENI** in the VPC — ensure NAT/route for internet-bound calls, e.g., to other AWS APIs).
- **Lake Formation**: Provides fine-grained (row/column/table-level) access control on top of the Glue Data Catalog, centralizing permissions across Athena, Redshift Spectrum, EMR, Glue.

---

## 10. Monitoring & Troubleshooting

- **CloudWatch Logs**: Job run logs, error logs (`/aws-glue/jobs/error` and `/aws-glue/jobs/output`).
- **CloudWatch Metrics**: DPU usage, memory profile, ETL data movement.
- **Glue Job Metrics / Spark UI**: Visualize stage-level performance, identify skew/bottlenecks.
- Common issues:
  * **OOM (Out of Memory) errors** → increase worker type (G.2X) or increase number of workers, or optimize partitioning/skew.
  * **Job running slow due to too many small files ("small file problem")** → use `groupFiles`/`groupSize` options when reading, or compact files upstream.
  * **Crawler creating too many tables from one S3 prefix** → inconsistent schemas/formats in the prefix; enforce consistent structure or use exclude patterns.
  * **Job bookmarks not working** → ensure bookmarks enabled at job creation, source supports it, and transformation context is consistent across runs.
  * **Job cost highly variable/unpredictable across runs** → consider **Glue Auto Scaling** to match worker count to actual per-stage demand instead of over-provisioning a fixed worker count.

---

## 11. Common Exam Scenarios & Gotchas

1. **Need to catalog data automatically from S3 for Athena queries** → use a **Glue Crawler** to populate the Data Catalog.
2. **New date-partitioned data must be queryable immediately with minimal overhead** → **Athena Partition Projection** instead of a Crawler.
3. **Incremental processing of only new files each run** → enable **Job Bookmarks**.
4. **Handling inconsistent/nested JSON schema** → use **DynamicFrame** with `ResolveChoice`/`Relationalize`.
5. **Convert raw CSV/JSON to Parquet for cost-efficient Athena queries** → **Glue ETL job** writing Parquet output.
6. **Analysts need no-code data cleaning** → **Glue DataBrew**.
7. **Need schema validation/compatibility enforcement for streaming producers/consumers** → **Glue Schema Registry**.
8. **Need to chain crawler → job → crawler with dependencies** → **Glue Workflows** with triggers.
9. **Fine-grained access control (column/row level) across Athena/Redshift Spectrum/EMR** → **Lake Formation** on top of Glue Data Catalog.
10. **Low-latency/lightweight script with no need for distributed Spark** → **Python Shell** job type.
11. **Cost optimization for non-time-sensitive batch jobs** → **Glue Flex** execution class.
12. **Cost optimization when resource needs vary a lot within one job's execution** → **Glue Auto Scaling** (dynamic worker count), not a larger fixed worker type.
13. **Automated data quality checks with pass/fail thresholds** → **Glue Data Quality** (DQDL rules).
14. **Deduplicate records that don't match exactly (typos, formatting differences)** → **FindMatches ML Transform**.
15. **Iteratively develop/test Spark ETL code with low cost before deploying as a job** → **Glue Interactive Sessions** (notebooks).
16. **Job failing with OOM / slow due to data skew** → adjust worker type/count, review partitioning strategy.

---

## 12. Quick Reference Cheat Sheet

- 1 DPU = **4 vCPU + 16 GB RAM**
- Worker types: **Standard, G.1X, G.2X, G.025X, G.4X, G.8X**
- Job Bookmarks → **incremental ETL**
- Glue Auto Scaling → **dynamic worker count during job execution**, based on Spark task parallelism
- DynamicFrame → **schema-flexible, semi-structured data**
- Crawler → **schema inference + Data Catalog population**
- Partition Projection → **skip the Crawler** when S3 partition layout is predictable
- FindMatches → **ML-based fuzzy deduplication/matching**
- Interactive Sessions → **notebook-based iterative Spark development**, billed per DPU-second
- Schema Registry formats: **Avro, JSON Schema**
- Data Quality language: **DQDL**
- Fine-grained access control: **Lake Formation**
- No-code prep tool: **DataBrew**
- Visual ETL authoring: **Glue Studio**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
