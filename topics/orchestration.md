# Data Pipeline Orchestration — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Orchestration ties ingestion, transformation, and loading steps into reliable, schedulable, dependency-aware pipelines. This is a core part of **Domain 1** (building pipelines) and shows up throughout **Domain 3** (operating/monitoring them).

| Service                     | Purpose                                                                  |
| ----------------------------- | --------------------------------------------------------------------------- |
| AWS Step Functions           | Serverless state-machine orchestration (visual workflow, JSON-defined)   |
| Amazon MWAA                  | Managed Apache Airflow — DAG-based orchestration, Python-defined         |
| Amazon EventBridge           | Event bus for scheduling and event-driven triggering across AWS services |
| AWS Lambda                   | Serverless compute — often the "glue" between orchestration and services |
| Glue Workflows               | Glue-native orchestration (crawlers/jobs/triggers) — see `glue.md`       |

---

## 2. AWS Step Functions

- **State machine**: Defined in **Amazon States Language (ASL, JSON)** — a sequence/graph of states (Task, Choice, Parallel, Wait, Map, etc.).
- **Standard vs. Express workflows**:
  * **Standard**: Long-running (up to 1 year), exactly-once execution, full execution history, pay-per-state-transition — good for auditable, long-duration pipelines.
  * **Express**: High-volume, short-duration (up to 5 min), at-least-once execution, pay-per-invocation/duration — good for high-throughput event processing (e.g., per-record workflows fed by Kinesis/S3 events).
- **Service integrations**: Direct, no-Lambda-needed integrations with Glue, EMR, Athena, Lambda, SNS/SQS, DynamoDB, ECS, and many others — reduces custom glue code.
- **Error handling**: Built-in `Retry`/`Catch` per state — declarative retry with backoff, without custom code.
- **Map state**: Iterate over an array (e.g., a list of S3 files) and run a sub-workflow per item, with configurable concurrency.
- Common exam answer for: *"orchestrate a multi-step pipeline (Glue job → validation → Athena query → notification) with visual monitoring and built-in retry logic, without managing servers."*

## 3. Amazon MWAA (Managed Airflow)

- Fully managed **Apache Airflow** — DAGs are defined in **Python**, giving maximum flexibility for complex dependency graphs, custom operators, and conditional branching logic.
- Choose MWAA over Step Functions when: the team already has **Airflow DAGs/expertise**, needs the broad **Airflow provider ecosystem** (hooks/operators for many third-party and on-prem systems), or requires complex scheduling semantics (e.g., backfills, SLAs, sensors) that map naturally to Airflow concepts.
- **Environment**: Runs on a managed set of workers/scheduler/webserver in your VPC; DAGs stored in an S3 bucket that MWAA polls.
- **Operators/Hooks**: Reusable Airflow components; AWS provides an Airflow provider package with hooks for Glue, EMR, Redshift, S3, etc.
- Higher operational cost/complexity than Step Functions — a common exam distractor is choosing MWAA for a simple linear pipeline where Step Functions would have less overhead.

## 4. Amazon EventBridge

- **Event bus**: Central router for events — default bus (AWS service events), custom buses (your own application events), partner buses (SaaS integrations).
- **Rules**: Match event patterns (or a **schedule expression** — cron/rate) and route matching events to targets (Lambda, Step Functions, SQS, SNS, Glue, etc.).
- **Scheduler**: EventBridge Scheduler provides more flexible, high-volume, one-time or recurring invocations than classic CloudWatch Events cron rules.
- Common role in pipelines: the **trigger layer** — e.g., "new object lands in S3 → EventBridge rule → starts Step Functions execution / Glue job," decoupling producers from orchestration logic.
- **Schema Registry** (EventBridge's own, distinct from Glue Schema Registry): Discovers and stores schemas of events flowing through a bus, generates code bindings.

## 5. AWS Lambda in Data Pipelines

- Common roles: lightweight **transformation** (Firehose/Kinesis data transform), **event-driven trigger** (S3 event → Lambda → downstream job), **glue code** between orchestration steps not covered by direct service integrations, **API-based ingestion** (behind API Gateway).
- **Event source mappings**: For polling-based sources (Kinesis, DynamoDB Streams, SQS) — Lambda polls and invokes in batches; configurable batch size, batching window, and **parallelization factor** (per shard, for Kinesis/DynamoDB Streams).
- **Concurrency**: Reserved concurrency (guarantee/cap capacity for a function) vs. provisioned concurrency (pre-warmed instances, reduces cold start) — relevant for latency-sensitive or high-throughput ingestion paths.
- **Timeout**: Max 15 minutes — a common reason to choose Glue Python Shell/Step Functions/EMR instead of Lambda for longer-running transformation logic.
- **Destinations**: On-failure/on-success routing (SQS, SNS, EventBridge, another Lambda) for asynchronous invocations — used for dead-letter-style error handling without custom code.

## 6. Choosing the Right Orchestration Tool (Common Exam Framing)

- **Simple linear/branching pipeline, want visual monitoring, minimal setup, native AWS service integrations** → **Step Functions**.
- **Complex DAGs, need Python-level control, team has existing Airflow knowledge, many external/third-party systems** → **MWAA**.
- **Event-driven triggering, scheduling, or decoupling producers from consumers across many services** → **EventBridge**.
- **Small, fast, stateless transformation or glue logic (<15 min)** → **Lambda**.
- **Orchestration is entirely within Glue (crawlers + ETL jobs)** → **Glue Workflows** (simplest option when nothing outside Glue is involved).

## 7. Common Exam Scenarios & Gotchas

1. **Multi-step pipeline across Glue, EMR, and Athena, need visual state tracking + built-in retries** → **Step Functions Standard workflow**.
2. **High-volume, short-lived, per-record workflows (e.g., fed by a stream)** → **Step Functions Express workflow**.
3. **Team has existing Airflow DAGs / needs complex Python-defined scheduling logic** → **MWAA**.
4. **Trigger a pipeline whenever a file lands in S3, decoupled from the orchestration logic itself** → **EventBridge rule → Step Functions/Lambda**.
5. **Run a lightweight transformation with a hard time limit under 15 minutes** → **Lambda**; beyond that, use **Glue Python Shell** or **Step Functions with Batch/EMR**.
6. **Reduce Lambda cold-start latency for a latency-sensitive stream processing function** → **Provisioned Concurrency**.
7. **Orchestration is fully contained within Glue crawlers/jobs, nothing external** → **Glue Workflows** (avoid overengineering with Step Functions).
8. **Need declarative retry/backoff on individual pipeline steps without custom code** → **Step Functions `Retry`/`Catch`**.

## 8. Quick Reference Cheat Sheet

- Step Functions: **JSON/ASL state machine**, Standard (long, exactly-once) vs Express (short, at-least-once)
- MWAA: **managed Airflow**, Python DAGs, best with existing Airflow expertise
- EventBridge: **event bus + scheduler**, decouples triggers from targets
- Lambda: **serverless compute**, 15-min max timeout, event source mappings for polling sources
- Simplest choice for Glue-only orchestration: **Glue Workflows**
- Provisioned Concurrency = **reduce Lambda cold starts**
- Reserved Concurrency = **cap/guarantee Lambda capacity**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
