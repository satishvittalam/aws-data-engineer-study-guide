# Amazon OpenSearch Service & Amazon QuickSight — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

These two services sit at the "consumption" end of the data pipeline — search/log analytics and business intelligence, respectively. They appear in **Domain 3: Data Operations & Support**, typically as destinations fed by Kinesis Firehose, Glue, or Athena.

| Service                  | Purpose                                                        |
| --------------------------- | ------------------------------------------------------------------ |
| Amazon OpenSearch Service | Managed search & analytics engine (fork of Elasticsearch)      |
| OpenSearch Dashboards     | Visualization layer for OpenSearch (fork of Kibana)              |
| Amazon QuickSight         | Serverless BI/dashboarding service                              |
| SPICE                     | QuickSight's in-memory data engine for fast dashboard performance |

---

## 2. Amazon OpenSearch Service

- **Purpose**: Full-text search, log analytics, and near real-time operational dashboards — not a general-purpose OLAP data warehouse (that's Redshift's job).
- **Domain**: A cluster of data nodes (+ optional dedicated master nodes) holding **indices** (analogous to tables), split into **shards** for parallelism.
- **Common ingestion paths**:
  * **Kinesis Data Firehose** → OpenSearch (built-in destination, no custom code).
  * **CloudWatch Logs subscription** → OpenSearch, for centralized log analytics.
  * Direct application writes via the OpenSearch REST/bulk API.
- **OpenSearch Serverless**: Auto-scaling option with no cluster/shard management — good for unpredictable search/log workloads, mirrors the "serverless" pattern seen across Redshift/EMR/MSK.
- **UltraWarm / Cold storage tiers**: Cost-tiering for OpenSearch data — hot (fast, expensive) → UltraWarm (S3-backed, cheaper, slightly slower) → Cold (archival) — analogous to S3 storage class tiering, but within OpenSearch itself.
- **Security**: Fine-grained access control (index/document/field-level), IAM-based domain access policies, VPC or public access (VPC strongly preferred for production), encryption at rest (KMS) and in transit.
- Common exam answer for: *"build a near real-time dashboard over application/clickstream logs with full-text search capability."*

## 3. Amazon QuickSight

- **Serverless BI**: No infrastructure to manage; pay-per-session or per-author pricing model.
- **SPICE (Super-fast, Parallel, In-memory Calculation Engine)**: QuickSight's in-memory data store — datasets imported into SPICE give fast dashboard performance without hitting the source system on every view; has per-account/per-user storage limits requiring periodic refresh scheduling for freshness.
- **Direct Query mode**: Alternative to SPICE — queries the source (Athena, Redshift, RDS, S3) live on each dashboard interaction; more current data, but slower and puts load on the source.
- **Data sources**: Athena, Redshift, RDS, S3 (via Athena/manifest files), OpenSearch, and many SaaS/third-party connectors.
- **ML Insights**: Built-in anomaly detection and forecasting on dashboard data without separate ML infrastructure.
- **Row-level security (RLS)**: Restrict which rows a given user/group can see within a shared dashboard — common exam answer for "different regional sales teams should only see their own region's data in the same dashboard."
- **Embedding**: Dashboards can be embedded into external applications (anonymous or registered-user embedding) via the QuickSight API/SDK.

## 4. Choosing Between Them (and Redshift/Athena)

- **Need full-text search or log/event exploration** → **OpenSearch Service**.
- **Need executive/business dashboards, scheduled reports, ML-driven forecasting on curated data** → **QuickSight**.
- **Need large-scale historical OLAP aggregation as the source data QuickSight/OpenSearch reads from** → **Redshift** or **Athena** (query-in-place on S3).
- These are typically the **last stage** of a pipeline that started with Kinesis/MSK/Glue and landed data in S3/Redshift/OpenSearch.

## 5. Common Exam Scenarios & Gotchas

1. **Build a near real-time dashboard over streaming clickstream/application logs with search capability** → **Kinesis Firehose → OpenSearch Service → OpenSearch Dashboards**.
2. **Give business users fast, interactive BI dashboards over Athena/Redshift data with minimal repeat query cost** → **QuickSight with SPICE**.
3. **Dashboard must always reflect the very latest data, even at query-time cost** → **QuickSight Direct Query mode** instead of SPICE.
4. **Different teams must see only their own data in a shared dashboard** → **QuickSight Row-Level Security (RLS)**.
5. **Reduce OpenSearch storage cost for older, less-frequently-queried log data** → **UltraWarm / Cold storage tiers**.
6. **Unpredictable search workload, want to avoid manual OpenSearch cluster/shard sizing** → **OpenSearch Serverless**.
7. **Embed a dashboard into a customer-facing application** → **QuickSight embedding (anonymous or registered)**.

## 6. Quick Reference Cheat Sheet

- OpenSearch = **search & log analytics**, not general OLAP
- OpenSearch storage tiers: **Hot → UltraWarm (S3-backed) → Cold**
- Common OpenSearch ingestion path: **Kinesis Firehose → OpenSearch**
- QuickSight SPICE = **in-memory cache**, fast but needs refresh scheduling
- QuickSight Direct Query = **always current**, slower, hits source live
- QuickSight RLS = **per-user/group row filtering** in a shared dashboard
- QuickSight ML Insights = **built-in anomaly detection/forecasting**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
