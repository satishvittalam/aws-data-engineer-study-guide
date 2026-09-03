# Amazon EMR — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Amazon EMR (Elastic MapReduce) is a managed big-data platform for running distributed frameworks — Spark, Hive, Presto/Trino, HBase, Flink — on clusters of EC2 (or serverless/EKS) capacity. It's the go-to for large-scale, custom, or framework-flexible processing in **Domain 1**.

| Deployment Option | Purpose                                                          |
| -------------------- | ------------------------------------------------------------------- |
| EMR on EC2          | Traditional cluster of EC2 instances you size and manage           |
| EMR Serverless      | Auto-scaling Spark/Hive jobs with no cluster management            |
| EMR on EKS          | Run EMR jobs on an existing Kubernetes (EKS) cluster                |
| EMR Studio          | Managed notebook environment (Jupyter) for interactive development |

---

## 2. Cluster Architecture (EMR on EC2)

- **Master node**: Manages the cluster, tracks task status, runs YARN ResourceManager/HDFS NameNode.
- **Core nodes**: Run tasks and store data in **HDFS** — losing a core node can mean data loss (HDFS-replicated data).
- **Task nodes**: Run tasks only, no HDFS storage — safe to use **Spot Instances** here since losing one doesn't lose data.
- **EMRFS**: EMR's connector allowing Hadoop/Spark to read/write directly to **S3** as if it were HDFS — the standard pattern is "S3 as the persistent data store, HDFS as ephemeral/scratch."

## 3. Instance Purchasing Strategy (Common Exam Topic)

- **On-Demand**: Master node and core nodes needing guaranteed availability.
- **Spot Instances**: Task nodes, or core nodes only when the workload tolerates interruption — significant cost savings for fault-tolerant, non-time-critical jobs.
- **Instance Fleets**: Mix instance types/purchase options within a node group for better Spot availability and cost optimization; more flexible than legacy Instance Groups.

## 4. Scaling

- **Managed Scaling**: EMR automatically adds/removes core and task nodes based on workload (YARN memory/pending tasks) — replaces manual/custom auto-scaling policies for most modern use cases.
- **EMR Serverless**: No cluster sizing at all — submit a Spark/Hive job, EMR provisions and scales workers automatically, scales to zero when idle. Common exam answer for "run periodic Spark jobs with variable size and least operational overhead."

## 5. Storage & Data Access

- **EMRFS (S3)**: Standard for reading/writing large datasets — supports consistent view options and works with lifecycle/storage classes.
- **HDFS**: Cluster-local, ephemeral by default (unless cluster is long-running) — good for intermediate/shuffle data, not for source-of-truth storage.
- **Local instance storage / EBS**: Root volumes and additional attached storage for temporary data.

## 6. Key Frameworks Supported

- **Apache Spark**: Most common — batch and structured streaming, PySpark/Scala.
- **Apache Hive**: SQL-on-Hadoop, uses the Glue Data Catalog (or Hive Metastore) for table definitions — can share a catalog with Athena/Redshift Spectrum.
- **Presto/Trino**: Fast interactive SQL across multiple data sources.
- **Apache HBase**: NoSQL wide-column store on HDFS/S3.
- **Apache Flink** (on EMR): Stream processing alternative to Managed Service for Apache Flink, for teams needing more cluster control.

## 7. Security

- **IAM roles**: Separate roles for the EMR service itself, EC2 instance profile (job execution), and (optionally) per-user/job runtime roles for fine-grained access.
- **Encryption**: At-rest (EBS/S3 via KMS), in-transit (TLS) — configured via **security configurations** attached to the cluster.
- **Kerberos**: Supported for strong authentication across multi-tenant clusters.
- **VPC**: Clusters launch inside a VPC/subnet — security groups control node-to-node and external access.
- **Lake Formation integration**: EMR (with runtime roles) can enforce Lake Formation's fine-grained permissions when querying the Glue Data Catalog.

## 8. Common Exam Scenarios & Gotchas

1. **Run periodic Spark ETL jobs with variable resource needs, least ops overhead** → **EMR Serverless**.
2. **Long-running cluster processing large-scale batch jobs, need cost control** → **On-Demand for master/core, Spot for task nodes**.
3. **Losing nodes must not lose data** → keep HDFS-dependent (core) nodes On-Demand; use **EMRFS/S3** as the durable store instead of relying on HDFS.
4. **Need Athena, Redshift Spectrum, and EMR to share one metadata source** → all point to the **Glue Data Catalog** (EMR via Hive Metastore compatibility).
5. **Interactive Spark development without spinning up a full cluster manually** → **EMR Studio** notebooks.
6. **Auto-scale cluster capacity based on job demand without custom scaling policies** → **EMR Managed Scaling**.
7. **Fine-grained (row/column) access control when querying via EMR** → **Lake Formation** with **EMR runtime roles**.
8. **Run EMR jobs on existing Kubernetes infrastructure instead of new EC2 clusters** → **EMR on EKS**.

## 9. Quick Reference Cheat Sheet

- Master = **coordination**; Core = **compute + HDFS storage**; Task = **compute only, Spot-safe**
- EMRFS = **S3 as durable storage** for Hadoop/Spark
- EMR Serverless = **no cluster management**, scales to zero
- Cost optimization: **Spot for task nodes**, On-Demand for master/core
- Shared catalog across EMR/Athena/Redshift Spectrum: **Glue Data Catalog**
- Managed Scaling = **automatic core/task node scaling** based on YARN metrics
- Fine-grained access on EMR: **Lake Formation + runtime roles**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
