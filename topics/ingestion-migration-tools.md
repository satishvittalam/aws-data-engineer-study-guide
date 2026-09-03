# Data Ingestion & Migration Tools — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Beyond streaming (Kinesis/MSK) and database replication (DMS), the exam tests a set of purpose-built tools for moving files/data into AWS from on-premises, edge, or external systems — part of **Domain 1**.

| Service              | Purpose                                                              |
| ---------------------- | ------------------------------------------------------------------------ |
| AWS DataSync          | Fast, automated online transfer of files between on-prem/cloud and AWS  |
| AWS Transfer Family   | Managed SFTP/FTPS/FTP servers backed by S3 or EFS                       |
| AWS Snow Family       | Physical devices for offline, large-scale, or disconnected data transfer |
| Amazon API Gateway    | Managed API front door — often used as a data ingestion entry point      |
| AWS AppFlow           | No-code integration for ingesting data from SaaS apps (Salesforce, etc.) |

---

## 2. AWS DataSync

- Purpose-built for **online, automated, large-scale file transfer** between on-premises storage (NFS, SMB), other clouds, or AWS storage services (S3, EFS, FSx) — and AWS-to-AWS transfers (e.g., S3 to S3 across accounts/regions).
- **Agent**: A virtual machine deployed on-prem (or in another cloud) that reads from the source and pushes data over the network — handles encryption in transit, scheduling, and bandwidth throttling.
- **Incremental transfers**: DataSync only copies changed/new data on subsequent runs (after the initial full copy) — efficient for ongoing sync, not just one-time migration.
- **Validation**: Automatic data integrity verification during and after transfer.
- Common exam answer for: *"regularly and automatically sync an on-premises file share into S3 for analytics, with minimal custom scripting."*

## 3. AWS Transfer Family

- Fully managed **SFTP, FTPS, and FTP(S)** servers, backed by **S3** or **Amazon EFS** as the storage layer — lets external partners/legacy systems keep using familiar file-transfer protocols while data lands directly in your data lake.
- **Identity**: Supports service-managed users, or integration with existing identity providers (Active Directory, custom IdP via Lambda) for authentication.
- Common exam answer for: *"a legacy partner system can only send data via SFTP, and files need to land in S3 without building/managing your own FTP server."*
- Distinct from DataSync: Transfer Family is for **external partners pushing data in via a file-transfer protocol**; DataSync is for **you actively pulling/syncing data from a known source**.

## 4. AWS Snow Family

- For **offline** or **extremely large-scale** data transfer where network transfer would be too slow or where connectivity is unreliable/absent.
- **Snowcone**: Small, portable — edge computing and small-scale data transfer (terabytes).
- **Snowball Edge**: Storage-optimized or compute-optimized variants — petabyte-scale transfer, can run compute/Lambda functions locally at the edge before shipping back.
- **Snowmobile**: Exabyte-scale — a shipping-container-sized mobile data center, for the largest migrations (datacenter shutdowns, etc.).
- Common exam answer for: *"migrate petabytes of data from an on-prem datacenter with limited/no reliable network bandwidth."*
- Devices are shipped to AWS, data is imported into S3 — encrypted with KMS at all times (device and data).

## 5. Amazon API Gateway (as an ingestion path)

- Managed REST/HTTP/WebSocket API front door — commonly used to accept data pushed by external applications or IoT devices directly into a pipeline (e.g., API Gateway → Lambda → Kinesis/DynamoDB/S3).
- **Direct service integrations**: API Gateway can integrate directly with Kinesis (`PutRecord`) or SQS without a Lambda in between, reducing latency/cost for simple pass-through ingestion.
- Throttling, usage plans, and API keys provide basic ingestion-side rate limiting/quota management.

## 6. AWS AppFlow

- No-code, managed integration service to ingest/sync data **between SaaS applications (Salesforce, ServiceNow, Zendesk, Slack, etc.) and AWS** (S3, Redshift) or other SaaS apps.
- Supports scheduled, event-driven, or on-demand flows; includes basic **field-level transformation and data validation** during transfer.
- Common exam answer for: *"regularly ingest data from Salesforce into S3/Redshift for analytics, without custom API integration code."*
- Often confused with DMS (databases) or Glue (general ETL) — AppFlow's distinguishing trait is **native, no-code SaaS application connectors**.

## 7. Common Exam Scenarios & Gotchas

1. **Regularly sync an on-prem NFS/SMB file share into S3, minimal custom code** → **AWS DataSync**.
2. **Partner system can only push data via SFTP** → **AWS Transfer Family** (backed by S3).
3. **Migrate petabytes of data with poor/no network connectivity** → **AWS Snow Family** (Snowball Edge for most cases, Snowmobile for exabyte scale).
4. **Ingest data directly from Salesforce/ServiceNow into S3 or Redshift without writing custom API integration** → **AWS AppFlow**.
5. **Accept data pushed by external apps/IoT devices over HTTPS directly into a stream** → **API Gateway → Kinesis** (direct integration, skip Lambda for simple pass-through).
6. **One-time historical migration vs. ongoing sync** → DataSync handles both (incremental after first run); Snow Family is generally for one-time bulk moves given bandwidth constraints.

## 8. Quick Reference Cheat Sheet

- DataSync = **online, automated, incremental** file transfer (on-prem ↔ AWS)
- Transfer Family = **managed SFTP/FTPS servers** backed by S3/EFS, for partner-initiated pushes
- Snow Family = **offline** transfer: Snowcone (small/edge) → Snowball Edge (PB-scale) → Snowmobile (EB-scale)
- API Gateway = **HTTP front door**, can integrate directly with Kinesis/SQS without Lambda
- AppFlow = **no-code SaaS-to-AWS integration** (Salesforce, ServiceNow, etc.)

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
