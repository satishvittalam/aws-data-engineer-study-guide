# Monitoring, Operations & Automation — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

**Domain 3: Data Operations & Support (22%)** tests your ability to keep pipelines running, diagnose failures, and automate deployment — this spans observability tooling, scripting/SDK usage, and infrastructure-as-code.

| Service/Concept     | Purpose                                                          |
| ---------------------- | --------------------------------------------------------------- |
| Amazon CloudWatch     | Metrics, logs, alarms, dashboards                                |
| AWS CloudTrail        | API-level audit logging (who did what, when)                    |
| AWS X-Ray             | Distributed tracing for request-level performance debugging      |
| AWS CloudFormation    | Infrastructure as Code (IaC) for AWS resources                   |
| AWS CDK               | IaC using general-purpose programming languages (compiles to CFN) |
| CodePipeline/CodeBuild | CI/CD for deploying pipeline code (Glue jobs, Lambda, etc.)      |
| boto3 / AWS SDKs      | Programmatic interaction with AWS services from application code |

---

## 2. Amazon CloudWatch

- **Metrics**: Numeric time-series data emitted by AWS services (e.g., Glue DPU usage, Kinesis `IteratorAge`, Lambda `Duration`/`Errors`/`Throttles`) — see each service's own study guide for its key metrics.
- **Alarms**: Trigger actions (SNS notification, Auto Scaling, Lambda) when a metric crosses a threshold over an evaluation period — the standard mechanism for **proactive pipeline failure detection**.
- **Logs**: Centralized log storage (`CloudWatch Logs`); **Log groups/streams**; **subscription filters** can stream logs in real time to Lambda, Kinesis, or OpenSearch for further processing/alerting.
- **Logs Insights**: Query language for ad-hoc analysis across log groups without exporting data elsewhere.
- **Dashboards**: Custom visual boards combining metrics from multiple services — used for pipeline health overviews.
- **Composite Alarms**: Combine multiple alarms with AND/OR logic to reduce noise (e.g., only page on-call if both ingestion lag AND error rate are elevated).
- **EventBridge vs. CloudWatch Events**: EventBridge is the evolution of CloudWatch Events for event routing (see `orchestration.md`); CloudWatch itself remains the metrics/logs/alarms service.

## 3. AWS CloudTrail

- Logs **API calls** made across your account — "who called what API, when, from where."
- **Management events**: Control-plane actions (creating/deleting resources) — logged by default.
- **Data events**: Object-level (S3 `GetObject`/`PutObject`) or table-level (DynamoDB) actions — higher volume, must be explicitly enabled, incurs additional cost.
- Common exam answer for: *"determine who deleted a specific S3 object or modified a Glue job"* → **CloudTrail** (not CloudWatch, which doesn't track *who* made API calls at that granularity).
- **CloudTrail Lake**: Managed data store for CloudTrail logs enabling SQL-based querying without building your own log pipeline.

## 4. AWS X-Ray

- **Distributed tracing**: Visualizes request flow and latency across multiple services (e.g., API Gateway → Lambda → DynamoDB) — helps pinpoint which component in a chain is the bottleneck or failure point.
- Less central to DEA-C01 than CloudWatch/CloudTrail, but can appear as the answer for *"trace a slow request across multiple serverless services to find the bottleneck."*

## 5. Infrastructure as Code

- **CloudFormation**: Declarative JSON/YAML templates defining AWS resources (Glue jobs, Kinesis streams, S3 buckets, IAM roles, etc.) — enables repeatable, version-controlled infrastructure deployment; supports **stacks**, **change sets** (preview changes before applying), and **drift detection** (identify manual changes made outside CFN).
- **AWS CDK**: Define infrastructure using a general-purpose language (Python, TypeScript, etc.), which synthesizes into CloudFormation — preferred when engineering teams want to use loops/conditionals/abstractions rather than raw YAML.
- Common exam answer for: *"deploy the same set of Glue jobs and IAM roles consistently across dev/test/prod environments"* → **CloudFormation (or CDK)**.
- **SAM (Serverless Application Model)**: CloudFormation extension simplifying Lambda/API Gateway/DynamoDB deployments — occasionally referenced for serverless-specific IaC.

## 6. CI/CD for Data Pipelines

- **CodePipeline**: Orchestrates the release process (source → build → test → deploy) across stages.
- **CodeBuild**: Managed build/test compute — e.g., packaging and validating Glue job scripts, running unit tests on PySpark code, or synthesizing a CDK app.
- **CodeDeploy / CodeCommit**: Less commonly tested for data pipelines specifically, but part of the same CI/CD family.
- Common exam framing: *"automatically test and deploy updated Glue ETL scripts whenever code is pushed to a repository"* → **CodePipeline + CodeBuild**, triggered by a source-stage webhook/EventBridge rule.

## 7. Programming Concepts Tested on the Exam

- **PySpark fundamentals**: Reading exam questions that show DataFrame transformations (`select`, `filter`, `groupBy`, `withColumn`, joins) and asking what the resulting output/schema would be — the exam tests **reading comprehension of Spark code**, not writing complex Spark from scratch.
- **boto3 (Python SDK)**: Recognizing correct usage patterns for common calls (e.g., `put_object`, `start_job_run` for Glue, `get_records` for Kinesis) — again, mostly reading/identifying correct vs. incorrect API usage in a snippet.
- **SQL**: Athena/Redshift-flavored SQL — window functions, CTEs, and query optimization concepts (predicate pushdown, partition pruning) are more heavily tested than exotic SQL syntax.
- **Idempotency**: A recurring theme — pipelines should produce the same result if a step is retried (e.g., using `INSERT OVERWRITE`/deterministic partitioning rather than blind `INSERT` on retry) to avoid duplicate data after failures.

## 8. Common Exam Scenarios & Gotchas

1. **Detect and alert when a pipeline stops processing (e.g., Kinesis consumer falls behind)** → **CloudWatch Alarm** on the relevant lag metric (e.g., `IteratorAge`), notifying via SNS.
2. **Determine who deleted a specific S3 object or Glue table** → **CloudTrail data events**, not CloudWatch.
3. **Reduce alert noise from multiple related alarms firing together** → **CloudWatch Composite Alarms**.
4. **Consistently deploy the same pipeline infrastructure across environments** → **CloudFormation/CDK**, not manual console setup.
5. **Automatically test/deploy updated Glue job code on every commit** → **CodePipeline + CodeBuild**.
6. **Trace a slow multi-service request to find the bottleneck** → **AWS X-Ray**.
7. **A retried pipeline step must not create duplicate data** → design for **idempotency** (e.g., `INSERT OVERWRITE`, deterministic keys, deduplication logic).
8. **Query historical API activity with SQL instead of building a custom log pipeline** → **CloudTrail Lake**.

## 9. Quick Reference Cheat Sheet

- CloudWatch = **metrics, logs, alarms, dashboards** (the "what's happening" service)
- CloudTrail = **API audit log** — "who did what" (management + optional data events)
- X-Ray = **distributed tracing** across services
- CloudFormation/CDK = **infrastructure as code**
- CodePipeline + CodeBuild = **CI/CD** for pipeline code
- Idempotency = **safe retries without duplicate data**
- Exam tests **reading** PySpark/boto3/SQL snippets more than writing complex code from scratch

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
