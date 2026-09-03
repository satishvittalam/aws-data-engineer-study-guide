# Amazon S3 & Data Lakes — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview
Amazon S3 is the foundational object storage service for data lakes on AWS. It underpins nearly every DEA-C01 domain — ingestion, storage, cataloging, transformation, and analytics all commonly land on or read from S3.

| Concept | Purpose |
|---|---|
| Bucket | Top-level container for objects, globally unique name, region-scoped |
| Object | File + metadata, stored with a key (full path) inside a bucket |
| Storage Class | Cost/performance tier for objects (Standard, IA, Glacier, etc.) |
| Lifecycle Policy | Automated transition/expiration rules for objects |
| Data Lake | Centralized repository (usually S3) storing structured/semi-structured/unstructured data at any scale |
| Lake Formation | Governance layer for building/securing data lakes on S3 + Glue Catalog |

---

## 2. S3 Core Concepts

- **Object**: Key (path), value (data, up to 5 TB), version ID (if versioning enabled), metadata, ACL/tags.
- **Bucket naming**: Globally unique, DNS-compliant, region-specific at creation.
- **Consistency**: S3 provides **strong read-after-write consistency** for all operations (PUTs and DELETEs), including overwrite PUTs and GETs.
- **Durability/Availability**: 11 nines durability (99.999999999%); availability varies by storage class (Standard ~99.99%).
- **Multipart Upload**: Required/recommended for objects >100 MB (mandatory above 5 GB); allows parallel part uploads and resumable transfers.
- **S3 Select / Glacier Select**: Retrieve a subset of data from an object using SQL-like expressions — reduces data transfer/cost for large CSV/JSON/Parquet files.

---

## 3. Storage Classes

| Class | Use Case | Retrieval | Min Duration | Notes |
|---|---|---|---|---|
| S3 Standard | Frequently accessed data | Immediate | None | Default, highest cost |
| S3 Intelligent-Tiering | Unknown/changing access patterns | Immediate | None | Auto-moves objects between tiers based on access, small monitoring fee |
| S3 Standard-IA | Infrequent access, rapid retrieval needed | Immediate | 30 days | Lower storage cost, retrieval fee |
| S3 One Zone-IA | Infrequent, non-critical/reproducible data | Immediate | 30 days | Single AZ — lower cost, less resilient |
| S3 Glacier Instant Retrieval | Archive with millisecond access | Immediate | 90 days | Rarely accessed but instant needed |
| S3 Glacier Flexible Retrieval | Archive, retrieval in minutes-hours | Minutes/hours | 90 days | Expedited (1-5 min), Standard (3-5 hr), Bulk (5-12 hr) |
| S3 Glacier Deep Archive | Long-term archive, lowest cost | 12-48 hours | 180 days | Cheapest storage class |

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

### Small File Problem
- Many small files → excessive overhead (S3 request costs, slower query planning, more Spark/Athena tasks).
- Mitigate via: **compaction jobs** (Glue/EMR), Firehose buffering before delivery, or `groupFiles`/`groupSize` in Glue DynamicFrame read options.

---

## 5. Security & Access Control

- **IAM Policies**: Identity-based permissions attached to users/roles.
- **Bucket Policies**: Resource-based JSON policies attached directly to a bucket (can grant cross-account access).
- **ACLs**: Legacy, object/bucket-level access control — generally discouraged in favor of IAM/bucket policies.
- **Block Public Access**: Account/bucket-level setting to prevent public exposure — enabled by default on new buckets.
- **S3 Access Points**: Named network endpoints with their own policy, simplifying access management for shared datasets/multi-tenant access.
- **VPC Endpoints (Gateway type for S3)**: Private connectivity from VPC to S3 without traversing the internet.
- **Encryption**:
  - **SSE-S3**: S3-managed keys (AES-256), default option.
  - **SSE-KMS**: AWS KMS-managed keys, supports auditing (CloudTrail) and key rotation policies, has request quota limits (KMS API throttling on high-throughput workloads — common exam gotcha).
  - **SSE-C**: Customer-provided keys, client manages key material.
  - **Client-side encryption**: Data encrypted before upload by the application.
- **Object Lock (WORM)**: Write-once-read-many — governance or compliance mode, used for regulatory retention requirements.

---

## 6. Data Lake Architecture on AWS

- **Zones/Layers** (common pattern, not an official AWS term but exam-relevant):
  - **Raw/Landing zone**: Ingested data as-is.
  - **Staging/Cleansed zone**: Validated, deduplicated, format-converted (e.g., Parquet).
  - **Curated/Analytics zone**: Aggregated, business-ready datasets for BI/ML.
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
4. **Too many small files causing slow queries** → **compaction** (Glue/EMR job) or buffer before writing (Firehose).
5. **Cross-account access to a Glue Catalog/S3 data lake with fine-grained permissions** → **AWS Lake Formation**.
6. **Automatically trigger processing when new data lands in S3** → **S3 Event Notifications** (Lambda/SQS/EventBridge) or **Glue trigger via Workflow**.
7. **High-throughput KMS-encrypted uploads throttled** → KMS request quota limits; consider **SSE-S3** or request a KMS quota increase, or use **S3 Bucket Keys** to reduce KMS API calls.
8. **Need to query subset of large CSV/JSON/Parquet object without full retrieval** → **S3 Select**.
9. **Disaster recovery / low-latency global read access to same data** → **Cross-Region Replication**.
10. **Retain records for compliance, prevent deletion/overwrite** → **S3 Object Lock (Compliance mode)**.
11. **Analyzing usage/cost patterns across many buckets/accounts** → **S3 Storage Lens**.
12. **Need private connectivity from VPC-based Glue jobs/EMR to S3 without internet** → **VPC Gateway Endpoint for S3**.

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
- Governance layer for data lake permissions: **Lake Formation**
- Query-in-place engines: **Athena, Redshift Spectrum, EMR**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
