# AWS Databases — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview
Database services fall primarily under **Domain 2: Data Store Management (26%)** of the DEA-C01 exam, with database migration/CDC topics also appearing in **Domain 1: Data Ingestion and Transformation (34%)**. A large share of scenario questions test your ability to **choose the right data store** for a given access pattern, latency requirement, and consistency need.

| Service | Type | Primary Use Case |
|---|---|---|
| Amazon RDS | Relational (managed) | Traditional OLTP workloads, structured data |
| Amazon Aurora | Relational (MySQL/PostgreSQL compatible) | High-performance OLTP, cloud-native scaling |
| Amazon DynamoDB | NoSQL (key-value/document) | Low-latency, high-throughput, flexible schema |
| Amazon Redshift | Data warehouse (columnar) | OLAP, analytics, large-scale aggregations |
| Amazon ElastiCache | In-memory cache | Sub-millisecond read latency, session storage |
| AWS DMS | Migration/replication service | DB migration, ongoing CDC replication |
| Amazon Neptune | Graph database | Relationship-heavy data (social, fraud) |
| Amazon Timestream | Time-series database | IoT, metrics, time-stamped data |
| Amazon DocumentDB | Document (MongoDB-compatible) | JSON document workloads |
| Amazon QLDB | Ledger database | Immutable, cryptographically verifiable transaction log |

---

## 2. Choosing the Right Data Store (High-Frequency Exam Topic)
- **Structured, relational, transactional (ACID)** → RDS/Aurora.
- **Key-value/document, need single-digit ms latency at scale, flexible schema** → DynamoDB.
- **Analytics/BI over large historical datasets, complex joins/aggregations** → Redshift.
- **Need sub-ms caching layer in front of a DB** → ElastiCache (Redis/Memcached).
- **Highly connected data, relationship traversal queries** → Neptune.
- **Time-stamped/IoT sensor data at scale** → Timestream.
- **Immutable audit trail, cryptographic verification** → QLDB.
- **MongoDB-compatible document workloads** → DocumentDB.
- **Query data directly in S3 without loading** → Athena/Redshift Spectrum (not a database, but a common distractor answer).

---

## 3. Amazon RDS & Aurora

### RDS Core Concepts
- Managed relational DB: MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Aurora.
- **Multi-AZ**: Synchronous standby replica in another AZ for **high availability/failover** (not for read scaling — common exam gotcha).
- **Read Replicas**: Asynchronous, used for **read scaling** and can be promoted to standalone instances; can be cross-region.
- **Automated backups**: Point-in-time recovery within retention window (1-35 days); snapshots persist beyond retention if manual.
- **RDS Proxy**: Connection pooling layer — reduces connection overhead for Lambda/serverless apps, improves failover time.
- **Storage autoscaling**: Automatically increases allocated storage when thresholds are hit.
- **Encryption**: KMS-based at-rest encryption; enabling requires it at creation time (cannot enable on existing unencrypted instance directly — must snapshot/copy/encrypt).

### Aurora Specifics
- Storage auto-scales up to 128 TB, decoupled from compute; 6 copies of data across 3 AZs.
- **Aurora Replicas**: Up to 15, low replication lag (faster than standard RDS read replicas).
- **Aurora Serverless v2**: Auto-scales capacity (ACUs) based on load — good for intermittent/unpredictable workloads.
- **Aurora Global Database**: Cross-region replication with <1 sec typical lag, for DR and low-latency global reads.
- **Backtrack**: Rewind Aurora MySQL cluster to a prior point in time without restoring from backup.

---

## 4. Amazon DynamoDB

### Core Concepts
- **Partition key** (required) determines data distribution; **sort key** (optional) enables range queries within a partition.
- **Primary key** = partition key (simple) or partition + sort key (composite).
- **Local Secondary Index (LSI)**: Same partition key, different sort key — must be created at table creation, shares provisioned throughput.
- **Global Secondary Index (GSI)**: Different partition/sort key — created/modified anytime, has its own throughput settings.
- **Capacity modes**:
  - **On-Demand**: Pay-per-request, auto-scales instantly — good for unpredictable traffic.
  - **Provisioned**: Set RCU/WCU, use Auto Scaling for gradual adjustment — cheaper for predictable, steady workloads.
- **DynamoDB Streams**: Captures item-level changes (INSERT/MODIFY/REMOVE), commonly triggers **Lambda** for event-driven processing/CDC pipelines.
- **DAX (DynamoDB Accelerator)**: In-memory cache, microsecond read latency, transparent to application (write-through cache).
- **TTL (Time to Live)**: Auto-expire/delete items based on timestamp attribute — reduces storage cost.
- **Consistency**: Eventually consistent reads (default, cheaper) vs strongly consistent reads (higher cost, up-to-date).
- **Transactions**: `TransactWriteItems`/`TransactGetItems` for ACID-compliant multi-item operations.

### Design Patterns
- **Single-table design**: Store multiple entity types in one table using generic PK/SK naming — reduces joins (which DynamoDB doesn't support natively).
- **Hot partition avoidance**: Choose high-cardinality partition keys to distribute load evenly; avoid sequential/low-cardinality keys (e.g., date-only).
- **Sparse indexes**: GSIs that only include items with a specific attribute present — efficient filtering.

---

## 5. Amazon Redshift

### Core Concepts
- **MPP (Massively Parallel Processing)** columnar data warehouse; optimized for large aggregations/OLAP, not transactional workloads.
- **Node types**: Leader node (query planning/coordination) + compute nodes (data storage/execution).
- **RA3 nodes**: Decouple compute and storage — scale independently, backed by Redshift Managed Storage (S3).
- **Distribution styles**:
  - **KEY**: Distribute rows by a column's hash — colocate related data for join performance.
  - **ALL**: Full copy of table on every node — best for small, frequently joined dimension tables.
  - **EVEN**: Round-robin distribution — default when no clear join pattern.
- **Sort keys**: Compound (ordered, good for range queries) vs Interleaved (equal weight across columns, better for varied query patterns but higher maintenance cost).
- **Redshift Spectrum**: Query data directly in S3 (external tables) without loading — joins S3 data with Redshift tables.
- **COPY / UNLOAD**: Bulk load from (COPY) and export to (UNLOAD) S3, DynamoDB, EMR — parallelized, much faster than individual INSERTs.
- **WLM (Workload Management)**: Queue-based resource allocation; **Automatic WLM** vs manual queue configuration for query prioritization.
- **Concurrency Scaling**: Automatically adds transient clusters to handle traffic spikes without manual intervention.
- **Vacuum & Analyze**: Reclaim space from deleted rows and update statistics for the query planner (needed since Redshift doesn't auto-vacuum by default in all cases).
- **Materialized Views**: Precomputed query results, refreshed periodically — speeds up repeated complex aggregations.

---

## 6. AWS Database Migration Service (DMS)

- **Purpose**: Migrate databases to AWS with minimal downtime; supports **homogeneous** (same engine) and **heterogeneous** (different engine, e.g., Oracle → Aurora) migrations.
- **Replication instance**: Compute resource running the migration/replication tasks.
- **Endpoints**: Source and target connection definitions.
- **Migration types**:
  - **Full load**: One-time bulk migration of existing data.
  - **Full load + CDC**: Bulk load, then continuous replication of ongoing changes (most common exam scenario for "keep source and target in sync").
  - **CDC only**: Replicate only ongoing changes (source already migrated by other means).
- **Schema Conversion Tool (SCT)**: Converts schema/code (stored procedures, views) when migrating between different engines — used alongside DMS for heterogeneous migrations.
- **Common target for data lakes**: DMS is frequently used to continuously replicate on-prem/RDS database changes into **S3** (as Parquet/CSV) for analytics — very common exam pattern.
- **Validation**: DMS can validate that source and target data match after migration.

---

## 7. Amazon ElastiCache

- **Redis**: Supports persistence, replication, Multi-AZ failover, pub/sub, complex data structures (sorted sets, etc.) — preferred when durability/HA matters.
- **Memcached**: Simple, multi-threaded, no persistence/replication — good for simple, ephemeral caching needs.
- **Caching strategies**:
  - **Lazy loading**: Load into cache only on cache miss — simple, but can serve stale data.
  - **Write-through**: Update cache whenever DB is updated — keeps cache fresh, adds write latency.
  - **TTL**: Combine with lazy loading to bound staleness.
- Common exam use case: Reduce read load on RDS/Aurora/DynamoDB for frequently accessed, rarely changing data.

---

## 8. Other Purpose-Built Databases

- **Amazon Neptune**: Graph database (Gremlin/openCypher/SPARQL) — fraud detection, recommendation engines, knowledge graphs.
- **Amazon Timestream**: Purpose-built time-series DB — automatic tiering (recent data in memory, older in magnetic storage), built-in time-series functions.
- **Amazon DocumentDB**: MongoDB-compatible document store — JSON-like documents, used when app is already MongoDB-based.
- **Amazon QLDB**: Immutable, append-only ledger with cryptographic verification — use when you need a verifiable transaction history (e.g., financial records), NOT for general-purpose transactional workloads.
- **Amazon Keyspaces**: Managed Apache Cassandra-compatible database — wide-column NoSQL.

---

## 9. Security & Governance for Databases

- **Encryption at rest**: KMS-managed keys across RDS, Aurora, DynamoDB, Redshift; DynamoDB encrypts by default.
- **Encryption in transit**: SSL/TLS connections enforced via parameter groups/connection settings.
- **IAM Database Authentication**: Use IAM roles/tokens instead of native DB passwords (supported on RDS MySQL/PostgreSQL, Aurora).
- **Secrets Manager**: Store/rotate DB credentials automatically — integrates with RDS for automatic rotation.
- **VPC Security Groups**: Control network-level access to DB instances.
- **DynamoDB fine-grained access control**: IAM policies with condition keys restrict access to specific items/attributes.
- **Redshift**: Supports column-level and row-level security, integrates with Lake Formation for cross-service governance.

---

## 10. Common Exam Scenarios & Gotchas
1. **Need HA failover for relational DB, not read scaling** → **Multi-AZ RDS/Aurora** (not read replicas).
2. **Need to scale read traffic for relational DB** → **Read Replicas** (can be cross-region for Aurora Global Database).
3. **Millisecond-latency NoSQL access at massive scale, unpredictable traffic** → **DynamoDB with On-Demand capacity**.
4. **Need to react to item-level changes in DynamoDB in real time** → **DynamoDB Streams + Lambda**.
5. **Reduce read latency further on DynamoDB without app changes** → **DAX**.
6. **Continuously sync on-prem/RDS database changes into a data lake (S3)** → **DMS with Full Load + CDC**.
7. **Migrating between different DB engines (e.g., Oracle → PostgreSQL)** → **DMS + Schema Conversion Tool (SCT)**.
8. **Query S3 data alongside Redshift tables without loading** → **Redshift Spectrum**.
9. **Speed up repeated complex aggregation queries in Redshift** → **Materialized Views**.
10. **Distribute large fact table efficiently for joins in Redshift** → **KEY distribution style** on the join column.
11. **Small, frequently joined dimension table in Redshift** → **ALL distribution style**.
12. **Reduce DB load for frequently-read, rarely-changed data** → **ElastiCache (Redis/Memcached) with lazy loading**.
13. **Need immutable, verifiable audit log of transactions** → **QLDB** (not RDS/DynamoDB).
14. **Graph/relationship-heavy queries (e.g., fraud detection)** → **Neptune**.
15. **Rotate DB credentials automatically without app downtime** → **Secrets Manager integration with RDS**.
16. **Access DB using IAM roles instead of static passwords** → **IAM Database Authentication**.

---

## 11. Quick Reference Cheat Sheet
- RDS Multi-AZ = **HA/failover**, not read scaling
- Read Replicas = **read scaling**, async replication
- Aurora storage: auto-scales to **128 TB**, 6 copies across 3 AZs
- DynamoDB GSI: own throughput, can add anytime | LSI: shares throughput, must define at table creation
- DynamoDB Streams + Lambda = **event-driven CDC pattern**
- DAX = **microsecond** read cache for DynamoDB
- Redshift distribution styles: **KEY** (join optimization), **ALL** (small dim tables), **EVEN** (default/no clear pattern)
- Redshift Spectrum = **query S3 data in place**
- DMS Full Load + CDC = **most common "keep in sync" migration pattern**
- SCT = used for **heterogeneous** engine migrations (schema conversion)
- ElastiCache: **Redis** (persistence/HA/replication) vs **Memcached** (simple, ephemeral)
- QLDB = **immutable ledger**, not general OLTP
- Neptune = **graph** database
- Timestream = **time-series** database

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
