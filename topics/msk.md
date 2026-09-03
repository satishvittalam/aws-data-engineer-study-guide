# Amazon MSK — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Amazon Managed Streaming for Apache Kafka (MSK) is a fully managed Apache Kafka service. It sits alongside Kinesis in **Domain 1: Data Ingestion & Transformation**, and the exam frequently tests **when to choose MSK over Kinesis**.

| Component        | Purpose                                                                 |
| ------------------ | -------------------------------------------------------------------------- |
| MSK Provisioned   | You manage broker count/type, storage, scaling                          |
| MSK Serverless    | Auto-scaling capacity, no broker/partition management                   |
| MSK Connect       | Managed Kafka Connect — connectors to/from S3, Redshift, OpenSearch, etc. |
| Broker            | A Kafka server that stores and serves data                              |
| Topic / Partition | Logical stream (topic) split into ordered partitions for parallelism     |

---

## 2. Core Concepts

- **Cluster**: A set of brokers running Kafka, spread across Multi-AZ for durability.
- **Topic**: Named stream of records; split into **partitions** for parallel throughput (analogous to Kinesis shards).
- **Producers/Consumers**: Standard Kafka client APIs (not KPL/KCL) — MSK is Kafka-API-compatible, so existing Kafka apps/tools work with minimal change.
- **Consumer Groups**: Kafka's native mechanism for parallel, coordinated consumption and offset tracking (vs. KCL/DynamoDB checkpointing in Kinesis).
- **Replication Factor**: Number of broker copies of each partition — for durability/HA.
- **ZooKeeper / KRaft**: Older clusters used Apache ZooKeeper for metadata management; newer MSK supports **KRaft mode** (no ZooKeeper dependency).

## 3. Provisioned vs. Serverless

- **MSK Provisioned**: You choose broker type/count, storage per broker, and manage partition/scaling decisions — more control, more operational overhead, needed when you require fine-grained tuning or specific Kafka versions/configs.
- **MSK Serverless**: Automatically provisions and scales capacity; you just create topics and set partition counts; billed per throughput — best for unpredictable workloads or teams wanting Kafka semantics with minimal ops.

## 4. MSK Connect

- Managed **Kafka Connect** runtime — deploy source/sink connectors without managing connect worker infrastructure.
- Common sink connectors: **S3 Sink**, **Redshift**, **OpenSearch**, **Debezium** (CDC source connector from databases into Kafka).
- Auto-scaling workers based on load.

## 5. Security

- **Encryption**: At-rest via KMS; in-transit via TLS between clients/brokers and inter-broker.
- **Authentication**: IAM access control, SASL/SCRAM, mutual TLS (mTLS) — IAM auth is the simplest AWS-native option and a common exam answer for "avoid managing separate Kafka credentials."
- **Authorization**: Kafka ACLs (fine-grained topic-level permissions) or IAM policies (when using IAM auth).
- **VPC-only**: MSK clusters live inside a VPC — private connectivity by default; no public endpoint unless explicitly configured.

## 6. Monitoring

- **CloudWatch metrics**: Broker CPU/disk/network, `UnderReplicatedPartitions`, `ActiveControllerCount`, consumer lag.
- **Open monitoring with Prometheus**: MSK can expose JMX/Node Exporter metrics for Prometheus-based monitoring — used by teams with existing Kafka observability tooling.

## 7. Kinesis vs. MSK (High-Frequency Exam Comparison)

| Factor              | Kinesis Data Streams                        | Amazon MSK                                          |
| --------------------- | ---------------------------------------------- | ------------------------------------------------------ |
| API                  | AWS-proprietary (KPL/KCL/SDK)                | Native Kafka API — drop-in for existing Kafka apps    |
| Management           | Fully AWS-managed, less to configure          | More configurable, more operational surface (Provisioned) |
| Best fit             | AWS-native apps, simpler ops, IAM-first       | Teams already using Kafka, need Kafka ecosystem tools |
| Scaling unit         | Shards                                        | Partitions                                            |
| Serverless option    | On-Demand mode                                | MSK Serverless                                        |
| Ecosystem            | AWS-native (Lambda, Firehose, KDA)            | Broad OSS Kafka ecosystem (Kafka Streams, ksqlDB, Connect) |

## 8. Common Exam Scenarios & Gotchas

1. **Team already has Kafka producers/consumers, migrating to AWS with minimal code change** → **MSK** (Kafka-API-compatible), not Kinesis.
2. **Need Kafka semantics but want zero broker/partition management** → **MSK Serverless**.
3. **CDC from a relational DB into a streaming pipeline** → **MSK Connect with Debezium connector** (or DMS into Kinesis, depending on architecture).
4. **Avoid managing separate Kafka credentials/ACLs** → **IAM access control** for MSK.
5. **Consumer falling behind** → check **consumer lag** metric (Kafka-native, analogous to Kinesis `IteratorAge`).
6. **Need private-only network access** → MSK is VPC-resident by default; use security groups + private subnets.

## 9. Quick Reference Cheat Sheet

- MSK = **managed Apache Kafka**, Kafka-API-compatible
- Scaling unit: **partitions** (Kafka) vs. **shards** (Kinesis)
- MSK Connect = **managed Kafka Connect** (S3, Redshift, OpenSearch, Debezium connectors)
- Simplest AWS-native auth: **IAM access control**
- Serverless option: **MSK Serverless**
- Choose MSK over Kinesis when: **existing Kafka investment / need Kafka ecosystem tools**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
