# AWS Kinesis — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview
Amazon Kinesis is a family of services for real-time streaming, ingestion, and processing of data at scale. Key services:

| Service | Purpose |
|---|---|
| Kinesis Data Streams (KDS) | Custom, real-time streaming data ingestion & processing |
| Kinesis Data Firehose | Fully managed delivery of streaming data to destinations (S3, Redshift, OpenSearch, Splunk, HTTP endpoints) |
| Kinesis Data Analytics (KDA) | Real-time analytics on streaming data using SQL or Apache Flink |
| Kinesis Video Streams | Ingest & process video/audio streams |

---

## 2. Kinesis Data Streams (KDS)

### Core Concepts
- **Stream**: Ordered sequence of data records, split into **shards**.
- **Shard**: Base throughput unit.
  - Write: 1 MB/sec or 1,000 records/sec per shard.
  - Read: 2 MB/sec per shard (shared across consumers) or 2 MB/sec per shard per consumer with **Enhanced Fan-Out (EFO)**.
- **Data Record**: Consists of a partition key, sequence number, and data blob (up to 1 MB).
- **Partition Key**: Determines which shard a record goes to (via hashing) — critical for avoiding "hot shards."
- **Sequence Number**: Unique, increasing identifier assigned per shard.
- **Retention Period**: Default 24 hours, extendable up to 365 days (extended retention incurs additional cost).

### Capacity Modes
- **Provisioned Mode**: You specify number of shards; scale manually or via API (UpdateShardCount) or resharding.
- **On-Demand Mode**: Automatically scales based on throughput; no shard management; billed per stream-hour + data throughput. Good for unpredictable workloads.

### Resharding
- **Shard Split**: Increase shard count (e.g., to handle hot shard / increase throughput).
- **Shard Merge**: Decrease shard count (combine two adjacent shards, e.g., low traffic).
- Resharding does not happen automatically in Provisioned mode.

### Producers
- **Kinesis Producer Library (KPL)**: Batches, compresses, retries records; adds slight delay (RecordMaxBufferedTime). Requires KCL or SDK to de-aggregate on consumer side.
- **AWS SDK (PutRecord/PutRecords)**: Simpler, synchronous, no batching benefits of KPL.
- **Kinesis Agent**: Monitors log files and sends data to KDS/Firehose.
- Other integrations: CloudWatch Events/EventBridge, IoT, API Gateway.

### Consumers
- **Kinesis Client Library (KCL)**: Java (and other languages via MultiLangDaemon) library for building consumer apps; handles load balancing across shards, checkpointing (via DynamoDB), and failure recovery.
- **AWS Lambda**: Can poll KDS as an event source (standard, shared throughput).
- **Kinesis Data Analytics / Apache Flink**: Direct stream processing.
- **Enhanced Fan-Out (EFO)**: Dedicated 2 MB/sec throughput PER CONSUMER per shard, uses HTTP/2 push model, lower latency (~70ms). Costs extra. Use when you have multiple consumers needing full throughput.
- Without EFO, consumers share the 2 MB/sec/shard read throughput (pull model, ~200ms latency).

### Security
- Encryption at rest: **KMS (server-side encryption)**.
- Encryption in transit: HTTPS endpoints.
- Access control via **IAM** policies.
- VPC endpoints (interface endpoints) supported for private connectivity.

### Monitoring & Troubleshooting
- CloudWatch metrics: `WriteProvisionedThroughputExceeded`, `ReadProvisionedThroughputExceeded`, `IteratorAge` (GetRecords.IteratorAgeMilliseconds — indicates consumer falling behind).
- **ProvisionedThroughputExceededException**: Occurs when producer/consumer exceeds shard limits — fix with better partition key distribution or resharding.

---

## 3. Kinesis Data Firehose

### Key Points
- Fully managed, **near real-time** (not truly real-time — has buffering, min ~60 sec).
- No shard management — auto scales.
- **Destinations**: S3, Redshift (via S3 staging), OpenSearch Service, Splunk, HTTP endpoints, third-party (Datadog, MongoDB, Snowflake, etc.).
- **Buffering**: Configurable buffer size (MBs) and buffer interval (seconds) — whichever threshold hits first triggers delivery.
- **Data Transformation**: Can invoke **Lambda** for transformation before delivery (e.g., format conversion, enrichment, decompression).
- **Format Conversion**: Can convert JSON to **Parquet/ORC** before storing in S3 — useful for optimizing downstream Athena/Redshift Spectrum queries.
- **Compression**: Supports GZIP, ZIP, Snappy (for S3 destination).
- **Error handling**: Failed records can be sent to an S3 error bucket (source record backup / error output prefix).
- Firehose can read directly from a **Kinesis Data Stream** (as a consumer) as its source, or receive data directly via PutRecord/PutRecordBatch API, or via Kinesis Agent, CloudWatch Logs/Events, IoT.

### Kinesis Data Streams vs Firehose (common exam comparison)
| Feature | KDS | Firehose |
|---|---|---|
| Latency | Real-time (~200ms) | Near real-time (~60 sec min) |
| Management | You manage shards/consumers (or on-demand) | Fully managed, serverless |
| Storage | Data stored (retention), replay possible | No storage/replay — straight-through delivery |
| Consumers | Custom consumers (KCL, Lambda, Flink) | Fixed destinations (S3, Redshift, OpenSearch, Splunk, HTTP) |
| Data Transform | Must build yourself | Built-in Lambda transform & format conversion |
| Use Case | Custom real-time apps, multiple consumers | Simple delivery/ETL pipeline to storage/analytics targets |

---

## 4. Kinesis Data Analytics (KDA)

- Two flavors:
  - **KDA for SQL**: Write SQL queries directly against streaming data (legacy, being phased toward Flink).
  - **KDA for Apache Flink** (Managed Service for Apache Flink): Java/Scala/Python/SQL applications using Apache Flink framework; supports complex event processing, windowing, stateful computations.
- Input sources: Kinesis Data Streams, Kinesis Data Firehose.
- Output destinations: KDS, Firehose (to S3/Redshift/etc.), Lambda.
- Use cases: real-time aggregations, anomaly detection (RANDOM_CUT_FOREST SQL function), streaming ETL, real-time dashboards.

---

## 5. Kinesis Video Streams
- Ingests video/audio/time-series data for ML, playback, batch processing.
- Integrates with Amazon Rekognition Video, SageMaker.
- Less emphasized in DEA-C01 exam but good to know it exists as part of Kinesis family.

---

## 6. Common Exam Scenarios & Gotchas
1. **"Hot shard" issue** → caused by poor partition key selection → fix: choose higher-cardinality partition key or reshard (split).
2. **Consumer falling behind** → check `IteratorAge` CloudWatch metric → add more shards, use EFO, or scale consumer application (more KCL workers).
3. **Need to convert streaming JSON to Parquet for Athena queries** → use **Firehose with format conversion** (not KDS directly).
4. **Need guaranteed ordering per key** → KDS preserves order within a shard (same partition key → same shard → ordered).
5. **Need multiple independent consumer applications reading full throughput** → use **Enhanced Fan-Out**.
6. **Cost-sensitive, unpredictable traffic** → use **On-Demand capacity mode**.
7. **Simple load into S3/Redshift/OpenSearch with minimal management** → use **Firehose**, not KDS + custom consumer.
8. **Data replay / multiple consumers reading historical data** → only possible with **KDS** (has retention), not Firehose (no storage).
9. **Real-time SQL aggregation on streaming data** → **Kinesis Data Analytics**.
10. **Lambda as KDS consumer** → Lambda polls shards; batch window & batch size configurable; supports parallelization factor for concurrent processing per shard.
11. Kinesis vs **Amazon MSK (Managed Streaming for Kafka)**: MSK = full Kafka API compatibility, more control, for teams already using Kafka; Kinesis = simpler, fully AWS-native, easier IAM integration.
12. Kinesis vs **SQS**: SQS = message queue (point-to-point, deleted after consumption); Kinesis = stream (multiple consumers can read same data, retained for replay).

---

## 7. Quick Reference Cheat Sheet
- Shard write limit: **1 MB/s or 1,000 records/s**
- Shard read limit (shared): **2 MB/s**
- Shard read limit (EFO, per consumer): **2 MB/s**
- Default retention: **24 hours**; max: **365 days**
- Max record size: **1 MB**
- Firehose min buffer interval: **60 seconds**
- Firehose supported transform: **Lambda**
- Firehose supported formats: **JSON → Parquet/ORC**
- KDA engines: **SQL** and **Apache Flink**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
