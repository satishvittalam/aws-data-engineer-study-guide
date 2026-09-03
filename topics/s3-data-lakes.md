# Amazon S3 & Data Lakes — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Amazon S3 is the foundational object storage service for data lakes on AWS. It underpins nearly every DEA-C01 domain — ingestion, storage, cataloging, transformation, and analytics all commonly land on or read from S3.

| Concept          | Purpose                                                                                               |
| ----------------- | ------------------------------------------------------------------------------------------------------ |
| Bucket           | Top-level container for objects, globally unique name, region-scoped                                  |
| Object           | File + metadata, stored with a key (full path) inside a bucket                                        |
| Storage Class    | Cost/performance tier for objects (Standard, IA, Glacier, etc.)                                       |
| Lifecycle Policy | Automated transition/expiration rules for objects                                                     |
| Data Lake        | Centralized repository (usually S3) storing structured/semi-structured/unstructured data at any scale |
| Open Table Format | Layer (Iceberg, Hudi, Delta Lake) adding ACID transactions, schema evolution, time travel to S3 data  |
| Lake Formation   | Governance layer for building/securing data lakes on S3 + Glue Catalog                                |

---

## 2. S3 Core Concepts

- **Object**: Key (path), value (data, up to 5 TB), version ID (if versioning enabled), metadata, ACL/tags.
- **Bucket naming**: Globally unique, DNS-compliant, region-specific at creation.
- **Consistency**: S3 provides **strong read-after-write consistency** for all operations (PUTs and DELETEs), including overwrite PUTs and GETs.
- **Durability/Availability**: 11 nines durability (99.999999999%); availability varies by storage class (Standard ~99.99%).
- **Multipart Upload**: Required/recommended for objects >100 MB (mandatory above 5 GB); allows parallel part uploads and resumable transfers.
- **S3 Select / Glacier Select**: Retrieve a subset of data from an object using SQL-like expressions — reduces data transfer/cost for large CSV/JSON/Parquet files.
- **Presigned URLs**: Time-limited URLs generated with a client's credentials that grant temporary GET/PUT access to a specific object without changing the bucket policy or IAM permissions of the recipient. Common exam answer for *"let an external/anonymous party upload or download a specific object for a limited time without giving them AWS credentials."*

### Versioning

- **Object Versioning**: When enabled on a bucket, every write creates a new version instead of overwriting — protects against accidental deletes/overwrites (a DELETE just adds a delete marker; the prior version is recoverable).
- **MFA Delete**: Optional extra protection requiring MFA to permanently delete a version or change versioning state — used for high-compliance buckets.
- **Noncurrent version lifecycle rules**: Transition or expire older versions automatically (e.g., delete noncurrent versions after 30 days) to control storage cost growth from versioning.
- Versioning is a **prerequisite for Cross-Region/Same-Region Replication** (see §7) and for restoring from Object Lock-protected objects.

---

## 3. Storage Classes

| Class                         | Use Case                                   | Retrieval     | Min Duration | Notes                                                                  |
| ------------------------------ | ------------------------------------------- | -------------- | ------------- | ------------------------------------------------------------------------ |
| S3 Standard                   | Frequently accessed data                   | Immediate     | None         | Default, highest cost                                                  |
| S3 Intelligent-Tiering        | Unknown/changing access patterns           | Immediate     | None         | Auto-moves objects between tiers based on access, small monitoring fee |
| S3 Standard-IA                | Infrequent access, rapid retrieval needed  | Immediate     | 30 days      | Lower storage cost, retrieval fee                                      |
| S3 One Zone-IA                | Infrequent, non-critical/reproducible data | Immediate     | 30 days      | Single AZ — lower cost, less resilient                                 |
| S3 Glacier Instant Retrieval  | Archive with millisecond access            | Immediate     | 90 days      | Rarely accessed but instant needed                                     |
| S3 Glacier Flexible Retrieval | Archive, retrieval in minutes-hours        | Minutes/hours | 90 days      | Expedited (1-5 min), Standard (3-5 hr), Bulk (5-12 hr)                 |
| S3 Glacier Deep Archive       | Long-term archive, lowest cost             | 12-48 hours   | 180 days     | Cheapest storage class                                                 |

### Lifecycle Policies

- Automate transitions between storage classes and expiration (deletion) based on object age/prefix/tag.
- Common pattern: Standard → IA (30 days) → Glacier (90 days) → Deep Archive (180 days) → Expire.
- Can also manage **incomplete multipart upload cleanup** and **noncurrent version expiration** (with versioning).

---

## 4. Data Organization for Data Lakes

### Partitioning

- Organize objects using key prefixes that mimic partition columns, e.g., `s3://bucket/table/year=2024/month=01/day=15/`.
- Enables **partition pruning** in Athena, Redshift Spectrum, Glue ETL — scans only relevant data, reduces cost/latency.
- Over-partitioning (too many small partitions) can hurt performance — balance partition granularity vs. file size.

### File Formats

- **Columnar (Parquet, ORC)**: Best for analytics — compression, predicate pushdown, column pruning; preferred for Athena/Redshift Spectrum/EMR.
- **Row-based (CSV, JSON)**: Easier to produce/human-readable but less efficient for large-scale analytics.
- **Avro**: Row-based, good for streaming/schema evolution use cases.
- Converting raw formats to Parquet (via Glue ETL/Firehose) is a very common exam scenario for cost/performance optimization.

### Open Table Formats & Amazon S3 Tables

- Plain Parquet/ORC files in S3 give you fast analytical reads but **no ACID transactions, no safe concurrent writes, and no built-in schema evolution or time travel** — updates/deletes typically require rewriting whole partitions.
- **Open table formats** (Apache Iceberg, Apache Hudi, Delta Lake) add a metadata/transaction layer on top of S3 objects to solve this:
  * **ACID transactions**: Safe concurrent reads/writes, atomic commits.
  * **Schema evolution**: Add/rename/drop columns without rewriting historical data.
  * **Time travel**: Query a table as of a prior snapshot/timestamp.
  * **Row-level UPDATE/DELETE/MERGE**: Enables CDC-style upserts directly on S3 data, without full-partition rewrites.
  * **Apache Iceberg** has the deepest native AWS integration — supported by Glue Data Catalog, Athena, EMR, and Redshift.
- **Amazon S3 Tables**: S3-native storage optimized specifically for Iceberg tables, with automatic file compaction, snapshot/partition management, and integration with the Glue Data Catalog (via SageMaker Lakehouse/Catalog) — reduces the operational overhead of self-managing Iceberg table maintenance.
- Common exam answer for: *"the data lake needs to support row-level updates/deletes and concurrent writers while remaining queryable by Athena/Redshift"* → **Iceberg (or S3 Tables) instead of raw Parquet.**

### Small File Problem

- Many small files → excessive overhead (S3 request costs, slower query planning, more Spark/Athena tasks).
- Mitigate via: **compaction jobs** (Glue/EMR), Firehose buffering before delivery, or `groupFiles`/`groupSize` in Glue DynamicFrame read options. Open table formats (Iceberg/S3 Tables) also handle compaction automatically as part of table maintenance.

---

## 5. Security & Access Control

- **IAM Policies**: Identity-based permissions attached to users/roles.
- **Bucket Policies**: Resource-based JSON policies attached directly to a bucket (can grant cross-account access).
- **ACLs**: Legacy, object/bucket-level access control — generally discouraged in favor of IAM/bucket policies (**Bucket owner enforced** setting disables ACLs entirely, the modern recommended default).
- **Block Public Access**: Account/bucket-level setting to prevent public exposure — enabled by default on new buckets.
- **S3 Access Points**: Named network endpoints with their own policy, simplifying access management for shared datasets/multi-tenant access.
- **S3 Multi-Region Access Points**: A single global endpoint that routes requests to the lowest-latency replicated bucket across regions, with failover — used for globally distributed applications reading/writing the same logical dataset.
- **VPC Endpoints (Gateway type for S3)**: Private connectivity from VPC to S3 without traversing the internet.
- **Encryption**:
  * **SSE-S3**: S3-managed keys (AES-256), default option.
  * **SSE-KMS**: AWS KMS-managed keys, supports auditing (CloudTrail) and key rotation policies, has request quota limits (KMS API throttling on high-throughput workloads — common exam gotcha).
  * **SSE-C**: Customer-provided keys, client manages key material.
  * **Client-side encryption**: Data encrypted before upload by the application.
- **Object Lock (WORM)**: Write-once-read-many — governance or compliance mode, used for regulatory retention requirements.

---

## 6. Data Lake Architecture on AWS

- **Zones/Layers** (common pattern, not an official AWS term but exam-relevant):
  * **Raw/Landing zone**: Ingested data as-is.
  * **Staging/Cleansed zone**: Validated, deduplicated, format-converted (e.g., Parquet or Iceberg).
  * **Curated/Analytics zone**: Aggregated, business-ready datasets for BI/ML.
- **Ingestion sources**: Kinesis, DMS, Glue, DataSync, Transfer Family, Snow Family (offline), direct application writes.
- **Cataloging**: Glue Data Catalog stores schema/metadata for all zones — enables Athena, Redshift Spectrum, EMR to query directly on S3.
- **Governance**: **AWS Lake Formation** centralizes permissions (row/column/tag-based access control) across the Glue Catalog + S3, replacing manual IAM/bucket policy management for fine-grained access.
- **Querying in place**: Athena (serverless SQL), Redshift Spectrum (SQL joins between Redshift + S3), EMR (Spark/Hive/Presto) — all read directly from S3 without needing to load/copy data first.

---

## 7. Replication & Cross-Region

- **S3 Cross-Region Replication (CRR)**: Async replication of objects to a bucket in another region — disaster recovery, latency reduction for global users, compliance.
- **S3 Same-Region Replication (SRR)**: Replication within the same region — log aggregation, prod/test environment sync.
- Requires **versioning enabled** on both source and destination buckets.
- Replication is **not retroactive** by default (only new objects after enabling, unless using S3 Batch Replication for existing objects).

---

## 8. Monitoring, Events & Automation

- **S3 Event Notifications**: Trigger Lambda, SQS, SNS, or EventBridge on object events (PUT, DELETE, restore complete) — common for event-driven ETL (e.g., new file lands → trigger Glue job/Lambda).
- **EventBridge integration**: More flexible routing/filtering of S3 events compared to native notifications.
- **S3 Storage Lens**: Organization-wide visibility into usage/activity metrics, cost optimization recommendations.
- **S3 Inventory**: Scheduled reports (CSV/ORC/Parquet) listing objects and metadata — useful for auditing large buckets without listing API calls.
- **S3 Batch Operations**: Run a single managed job (copy, tag, restore from Glacier, invoke Lambda, ACL change, Object Lock retention) across **millions of objects**, using an S3 Inventory report or manifest as the object list. Common exam answer for *"apply a one-time bulk change (e.g., re-tag or restore) across an entire large bucket with minimal custom code."*
- **CloudTrail**: Logs S3 API-level data events (object-level) and management events (bucket-level) for auditing.
- **Requester Pays**: Bucket owner designates that the requester pays for data transfer/request costs — used for shared public datasets.

---

## 9. Performance Optimization

- **Request rate**: S3 supports very high request rates per prefix (3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD per second per prefix) — use **varied key prefixes** to parallelize and avoid throttling on high-throughput workloads.
- **S3 Transfer Acceleration**: Uses CloudFront edge locations to speed up uploads over long distances.
- **Byte-range fetches**: Parallel download of large objects in parts for faster retrieval.
- **Multipart upload** improves throughput/resilience for large object uploads.

---

## 10. Common Exam Scenarios & Gotchas

1. **Reduce storage cost for infrequently accessed data with unpredictable patterns** → **S3 Intelligent-Tiering**.
2. **Long-term archival with lowest cost, retrieval time not critical** → **S3 Glacier Deep Archive**.
3. **Need queries to scan less data / reduce Athena cost** → **partition data + convert to Parquet/ORC**.
4. **Data lake needs row-level updates/deletes and concurrent writers, still queryable by Athena/Redshift** → **Apache Iceberg / Amazon S3 Tables**, not raw Parquet.
5. **Too many small files causing slow queries** → **compaction** (Glue/EMR job, or automatic via Iceberg/S3 Tables) or buffer before writing (Firehose).
6. **Cross-account access to a Glue Catalog/S3 data lake with fine-grained permissions** → **AWS Lake Formation**.
7. **Automatically trigger processing when new data lands in S3** → **S3 Event Notifications** (Lambda/SQS/EventBridge) or **Glue trigger via Workflow**.
8. **High-throughput KMS-encrypted uploads throttled** → KMS request quota limits; consider **SSE-S3** or request a KMS quota increase, or use **S3 Bucket Keys** to reduce KMS API calls.
9. **Need to query subset of large CSV/JSON/Parquet object without full retrieval** → **S3 Select**.
10. **Grant a third party temporary access to a specific object without issuing AWS credentials** → **Presigned URL**.
11. **Disaster recovery / low-latency global read access to same data** → **Cross-Region Replication** (or **Multi-Region Access Points** for a single routed endpoint).
12. **Retain records for compliance, prevent deletion/overwrite** → **S3 Object Lock (Compliance mode)**.
13. **Bulk one-time operation (copy/tag/restore) across millions of existing objects** → **S3 Batch Operations**.
14. **Protect against accidental overwrite/delete of objects, need ability to recover prior versions** → **S3 Versioning** (optionally + **MFA Delete** for compliance-grade protection).
15. **Analyzing usage/cost patterns across many buckets/accounts** → **S3 Storage Lens**.
16. **Need private connectivity from VPC-based Glue jobs/EMR to S3 without internet** → **VPC Gateway Endpoint for S3**.

---

## 11. Quick Reference Cheat Sheet

- Max object size: **5 TB**; multipart required above **5 GB** (recommended above 100 MB)
- Durability: **11 nines** (99.999999999%)
- Consistency: **Strong read-after-write** for all operations
- Default encryption option: **SSE-S3**
- Storage classes (cheap→expensive on retrieval speed tradeoff): **Deep Archive < Glacier Flexible < Glacier Instant < One Zone-IA < Standard-IA < Intelligent-Tiering < Standard**
- Minimum storage duration (IA/Glacier classes): **30 days (IA), 90 days (Glacier), 180 days (Deep Archive)**
- Request rate per prefix: **3,500 write / 5,500 read per second**
- Replication requires: **Versioning enabled**
- Presigned URL = **temporary access without changing IAM/bucket policy**
- S3 Batch Operations = **bulk action across millions of existing objects** (needs Inventory/manifest)
- Open table formats (Iceberg/Hudi/Delta) = **ACID + schema evolution + time travel** on top of S3 data
- Amazon S3 Tables = **S3-native, auto-managed Iceberg storage**
- Governance layer for data lake permissions: **Lake Formation**
- Query-in-place engines: **Athena, Redshift Spectrum, EMR**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
