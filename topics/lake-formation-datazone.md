# AWS Lake Formation & Amazon DataZone — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

These are the two dedicated **data governance** services for **Domain 4: Data Security & Governance**. Both are referenced throughout `glue.md`, `s3-data-lakes.md`, and `databases.md`, but deserve focused treatment since they're directly testable services in their own right.

| Service          | Purpose                                                                  |
| ------------------ | ----------------------------------------------------------------------- |
| AWS Lake Formation | Centralized, fine-grained permissions layer for the data lake (Glue Catalog + S3) |
| Amazon DataZone    | Data catalog, discovery, and governed data-sharing across an organization |

---

## 2. AWS Lake Formation

### Core Purpose

- Solves the problem of managing **fine-grained (database/table/column/row-level) permissions** across many consumers (Athena, Redshift Spectrum, EMR, Glue, QuickSight) without hand-maintaining a sprawling set of IAM/bucket policies per consumer.
- Sits **on top of** the Glue Data Catalog and S3 — it doesn't replace them, it adds a governed permissions layer.

### Key Concepts

- **Data Lake Locations**: S3 paths registered with Lake Formation — once registered, Lake Formation (not raw IAM/bucket policies) governs access to data under that path for integrated services.
- **Permissions model**: Grant/revoke permissions (`SELECT`, `DESCRIBE`, `ALTER`, `DROP`, `INSERT`) on **databases, tables, columns**, granted to **IAM principals**, at a much finer grain than IAM alone supports.
- **Row-level and cell-level security**: Achieved via **data filters** — define a filter (row predicate + column list) and grant it to specific principals, so different users querying the same table see different rows/columns.
- **LF-Tags (Tag-Based Access Control, TBAC)**: Attach key-value tags to databases/tables/columns, then grant permissions **based on tags** rather than naming every resource individually — scales much better than per-resource grants as the catalog grows (e.g., tag `confidentiality=PII`, grant `PII` access only to specific roles).
- **Blueprints**: Pre-built workflows (Glue Workflows under the hood) for common ingestion patterns (database snapshot/incremental, log ingestion) that automatically register data with Lake Formation as it lands.
- **Cross-account data sharing**: Lake Formation supports granting access to specific tables/databases to **another AWS account**, without that account needing broad S3/Glue Catalog access — the account-to-account governance layer for shared data lakes.
- **Credential vending**: When an integrated query engine (Athena, Redshift Spectrum, EMR with runtime roles) requests data, Lake Formation vends **temporary, scoped-down credentials** reflecting exactly what that principal is permitted to see — the enforcement mechanism behind the permissions model.

### Common Exam Scenarios

1. **Different analyst teams should see different columns of the same table (e.g., mask SSN for one group)** → **Lake Formation column-level permissions / data filters**.
2. **Grant access to hundreds of tables tagged as "public" without listing each one individually** → **LF-Tags (TBAC)**.
3. **Share specific tables with a partner's AWS account without broad IAM/bucket policy sharing** → **Lake Formation cross-account grants**.
4. **Automate onboarding a new data source into a governed data lake with minimal manual setup** → **Lake Formation Blueprints**.
5. **EMR/Athena queries should automatically enforce the same fine-grained permissions as everywhere else** → **Lake Formation-integrated query engines** using vended temporary credentials.

---

## 3. Amazon DataZone

### Core Purpose

- A **business data catalog and governed self-service data marketplace** — helps people across an organization **discover, understand, and request access to data**, distinct from Lake Formation's job of *enforcing* technical permissions.
- Think of it as the "storefront/catalog" layer; Lake Formation (and other integrated sources) remains the underlying enforcement mechanism.

### Key Concepts

- **Domains**: A DataZone Domain represents a business unit or organizational boundary for governance and data assets.
- **Projects**: Workspaces where users collaborate, publish, and consume data assets within a domain.
- **Data assets & catalog**: Published datasets (tables, dashboards, ML models) with business metadata, descriptions, glossaries — searchable by non-technical users.
- **Business glossaries**: Shared vocabulary/definitions attached to data assets, so a "customer_id" column means the same thing across teams.
- **Subscription workflow**: Users **request access** to a published data asset; owners **approve/deny** — a governed, auditable self-service access request flow, rather than engineers manually granting IAM/Lake Formation permissions on request.
- **Data sources**: Integrates with Glue Data Catalog, Redshift, S3 — automatically publishes/catalogs technical metadata alongside the business layer.
- Common exam answer for: *"enable business users across departments to discover and request access to datasets they don't own, with an approval workflow, without engineers manually handling every access request."*

## 4. Lake Formation vs. DataZone (Key Distinction)

| Aspect         | Lake Formation                                    | DataZone                                             |
| ---------------- | ---------------------------------------------------- | ------------------------------------------------------- |
| Primary role    | **Enforces** fine-grained technical permissions     | **Catalogs & governs discovery/access requests**       |
| Audience        | Data engineers/administrators                       | Business users, analysts, data producers/consumers      |
| Grain           | Database/table/column/row permissions                | Business assets, glossaries, subscription workflows      |
| Relationship    | Often the enforcement layer *behind* a DataZone grant | Can trigger/manage the actual permission grants underneath |

## 5. Quick Reference Cheat Sheet

- Lake Formation = **fine-grained permission enforcement** (table/column/row) on Glue Catalog + S3
- LF-Tags (TBAC) = **tag-based grants**, scales better than per-resource grants
- Data filters = **row/column-level security** in Lake Formation
- Blueprints = **pre-built ingestion workflows** that auto-register data with governance
- DataZone = **business catalog + self-service subscription/approval workflow**
- DataZone Domains/Projects = **organizational and collaboration boundaries**
- DataZone finds/requests data; Lake Formation **enforces** whether the request is actually allowed

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
