# IAM, KMS & Secrets Management — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

**Domain 4: Data Security & Governance (18%)** centers on identity, encryption, and credential management — foundational knowledge referenced throughout every other domain (every service's "Security" section points back here).

| Service                    | Purpose                                                        |
| ----------------------------- | ------------------------------------------------------------------ |
| AWS IAM                      | Identity and access management — users, roles, policies          |
| AWS KMS                      | Managed encryption key service                                    |
| AWS Secrets Manager          | Store, retrieve, and automatically rotate secrets (DB credentials, API keys) |
| AWS Systems Manager Parameter Store | Store configuration values and secrets (simpler, cheaper than Secrets Manager) |

---

## 2. IAM Core Concepts

- **Users**: Individual identities (people or applications) with long-term credentials — generally discouraged for application/service access in favor of roles.
- **Roles**: Assumable identities with temporary credentials — the standard way AWS services (Glue, Lambda, EMR, EC2) access other AWS resources.
- **Policies**: JSON documents defining permissions.
  * **Identity-based policies**: Attached to users/roles/groups.
  * **Resource-based policies**: Attached directly to a resource (S3 bucket policy, Glue Catalog resource policy, KMS key policy) — enable **cross-account access** without the accessing principal needing a role in the resource's account.
- **Policy evaluation logic**: Explicit **Deny** always wins; without an explicit Allow, access is denied by default; for cross-account access, **both** the identity-based policy (caller's account) and the resource-based policy (resource's account) must allow the action.
- **Condition keys**: Fine-grained restrictions within a policy (e.g., `aws:SourceIp`, `s3:ExistingObjectTag`, `dynamodb:LeadingKeys` for row-level DynamoDB access).
- **IAM Access Analyzer**: Identifies resources shared with external entities — helps catch unintended public/cross-account exposure.
- **Permissions boundaries**: Cap the maximum permissions a role/user can ever have, even if their attached policies grant more — used to safely delegate role creation to teams.

## 3. AWS KMS

- **Customer Master Keys (CMKs) / KMS keys**: Root encryption keys, never leave KMS unencrypted.
  * **AWS managed keys**: Created/managed by AWS for a specific service (e.g., `aws/s3`) — no key policy customization.
  * **Customer managed keys (CMKs)**: You create and control the key policy, rotation, and can enable **auditing of every key usage via CloudTrail**.
- **Envelope encryption**: KMS encrypts a **data key**, not the data itself directly, for large objects — the data key encrypts the actual data locally, then the (small) encrypted data key is stored alongside it; this is how S3/EBS/RDS encryption works under the hood and why encrypting large datasets doesn't require sending all the data through KMS.
- **Key rotation**: Automatic annual rotation available for customer-managed keys (the underlying key material changes, but the key ID/ARN stays the same, so existing encrypted data remains decryptable).
- **Key policies**: The primary access-control mechanism for a KMS key (a resource-based policy) — IAM policies alone are not sufficient to grant key usage; the key policy must also allow it (or explicitly delegate to IAM policies via a statement granting IAM control).
- **Request quotas / throttling**: High-throughput encryption workloads (e.g., many small S3 PUTs with SSE-KMS) can hit **KMS API request quotas** — mitigated with **S3 Bucket Keys** (reduces the number of KMS calls by generating a bucket-level data key reused for multiple objects) — a recurring exam gotcha, referenced in `s3-data-lakes.md`.
- **Multi-Region keys**: Replicate a KMS key across regions with the same key material — used for encrypting data that's replicated cross-region (e.g., with S3 CRR or Aurora Global Database) without needing to re-encrypt in the destination region.

## 4. AWS Secrets Manager

- Stores secrets (DB credentials, API keys, tokens) with **automatic rotation** support — has native integration to rotate **RDS/Aurora/Redshift/DocumentDB** credentials on a schedule using a built-in Lambda rotation function.
- Applications retrieve secrets at runtime via the API/SDK — avoids hardcoding credentials in code or environment variables.
- Costs more per-secret than Parameter Store, but rotation and fine-grained resource policies are the differentiators.
- Common exam answer for: *"rotate database credentials automatically without any application downtime or manual intervention."*

## 5. AWS Systems Manager Parameter Store

- General-purpose **configuration and secrets storage** — hierarchical key-value store (`/app/prod/db-host`).
- **Standard tier**: Free, up to 4 KB per parameter.
- **Advanced tier**: Larger values, higher throughput, supports parameter policies (e.g., expiration).
- **SecureString** parameter type: Encrypted via KMS — can store secrets, but **without Secrets Manager's automatic rotation** unless you build custom rotation logic.
- Choose Parameter Store over Secrets Manager when: cost-sensitive, don't need automatic rotation, or storing general (non-credential) configuration values alongside secrets in one hierarchical namespace.

## 6. IAM Database Authentication (cross-reference)

- Covered in `databases.md` §9 — allows RDS/Aurora connections using short-lived IAM-generated auth tokens instead of static passwords, removing the need to manage/rotate DB passwords at all for supported engines (MySQL, PostgreSQL, Aurora).

## 7. Common Exam Scenarios & Gotchas

1. **Grant a role in another AWS account access to your S3 bucket/Glue Catalog** → **resource-based policy** (bucket policy / Glue resource policy), not just an identity-based policy.
2. **High-throughput S3 uploads with SSE-KMS hitting request throttling** → enable **S3 Bucket Keys** to reduce KMS API call volume.
3. **Rotate RDS/Aurora credentials automatically with zero app downtime** → **Secrets Manager** with native rotation.
4. **Store simple, low-cost configuration values and secrets together, no rotation needed** → **Parameter Store (SecureString)**.
5. **Cap the maximum permissions a delegated team can grant themselves via IAM** → **Permissions boundaries**.
6. **Audit every time a specific encryption key was used** → **Customer-managed KMS key + CloudTrail**, not an AWS-managed key.
7. **Encrypt data consistently across regions after cross-region replication, without re-encrypting** → **KMS Multi-Region keys**.
8. **Access RDS/Aurora without managing static DB passwords at all** → **IAM Database Authentication**.
9. **Identify accidentally public or externally-shared resources** → **IAM Access Analyzer**.

## 8. Quick Reference Cheat Sheet

- Cross-account access → needs **resource-based policy** on the resource side (not just identity-based on the caller side)
- Explicit **Deny always wins**; default is **implicit deny**
- KMS envelope encryption: **KMS encrypts a data key, not the data directly**
- S3 Bucket Keys → **fewer KMS API calls**, fixes SSE-KMS throttling at scale
- Secrets Manager → **automatic rotation**, higher cost
- Parameter Store (SecureString) → **cheaper, no built-in rotation**
- Multi-Region KMS keys → **same key material across regions**, no re-encryption needed
- Permissions boundary → **caps max permissions**, doesn't grant them

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
