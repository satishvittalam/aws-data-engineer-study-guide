# VPC Networking for Data Engineers — Data Engineer Associate (DEA-C01) Study Guide

## 1. Overview

Networking is foundational, low-frequency-but-recurring content across **Domain 4** (and touches Domains 1–3 wherever a Glue job, EMR cluster, RDS instance, or MSK cluster needs to reach data privately). The exam doesn't test deep networking design, but expects you to recognize the right networking primitive for a given data-access scenario.

| Concept                | Purpose                                                                |
| ------------------------- | --------------------------------------------------------------------- |
| VPC                      | Isolated virtual network within a region                              |
| Subnet                   | A range of IPs within a VPC, tied to one AZ (public or private)       |
| Security Group           | Stateful, instance/ENI-level firewall (allow rules only)               |
| Network ACL (NACL)       | Stateless, subnet-level firewall (allow **and** deny rules)            |
| VPC Endpoint (Gateway)   | Private route to S3/DynamoDB, no internet/NAT required                |
| VPC Endpoint (Interface) | Private ENI-based connection to most other AWS services (PrivateLink) |
| NAT Gateway              | Allows private-subnet resources outbound internet access               |

---

## 2. Core Building Blocks

- **VPC**: A logically isolated network you define (CIDR range) within a region; most data services (RDS, Redshift, MSK, EMR, Glue connections, Lambda-in-VPC) are launched inside subnets of a VPC.
- **Public vs. Private subnets**: Public subnets have a route to an **Internet Gateway**; private subnets don't — data stores (RDS, Redshift, MSK brokers) should almost always live in **private subnets**, with access controlled via security groups and, where needed, a bastion/VPN/Direct Connect for admin access.
- **Security Groups**: **Stateful** — if inbound traffic is allowed, the matching outbound response is automatically allowed. Attached to ENIs/instances (e.g., an RDS instance's security group). Only support **allow** rules (implicit deny for everything not explicitly allowed).
- **Network ACLs**: **Stateless** — inbound and outbound rules must both be explicitly configured. Applied at the **subnet** level, evaluated in rule-number order, and can include **explicit Deny** rules — useful for blocking specific IP ranges at the subnet boundary regardless of security group config.

## 3. VPC Endpoints (High-Frequency Exam Topic)

- **Gateway Endpoints**: Free, route-table-based, private connectivity to **S3 and DynamoDB only** — the standard way for Glue jobs/EMR/Lambda-in-VPC to reach S3 without traversing the public internet or needing a NAT Gateway.
- **Interface Endpoints (AWS PrivateLink)**: ENI-based, billed per hour + data processed, private connectivity to most other AWS services (Glue, Kinesis, Secrets Manager, KMS, SNS, SQS, Athena, STS, CloudWatch, etc.) from within a VPC — required when a private-subnet resource (e.g., a Lambda function with no NAT) needs to call these APIs privately.
- Common exam answer for: *"a Glue job running in a private subnet with no NAT Gateway needs to write to S3 and also call Secrets Manager to retrieve DB credentials"* → **S3 Gateway Endpoint** (free, for S3) **+ Secrets Manager Interface Endpoint** (for the API call).

## 4. NAT Gateway vs. VPC Endpoints

- **NAT Gateway**: Lets private-subnet resources reach the **public internet** (e.g., calling a third-party API, downloading a package) — routes through an Internet Gateway in a public subnet, incurs hourly + data processing cost.
- **VPC Endpoints**: Keep traffic to **AWS services** entirely within the AWS network — cheaper (Gateway endpoints are free) and more secure than routing AWS-service traffic out through a NAT Gateway and back in.
- Exam framing: *"Reduce cost/improve security for a Glue job's traffic to S3"* → replace **NAT Gateway routing** with a **Gateway Endpoint**, since S3 traffic never needs to leave the AWS network.

## 5. Connectivity to On-Premises / Other Networks

- **VPN (Site-to-Site)**: Encrypted tunnel over the public internet between on-prem and a VPC — lower cost, higher latency/variable bandwidth, good for smaller/less latency-sensitive data transfer (e.g., DMS replication from an on-prem DB over VPN).
- **AWS Direct Connect**: Dedicated, private network connection between on-prem and AWS — higher bandwidth, lower/consistent latency, higher cost/setup effort; used for large-scale, latency-sensitive, or high-volume ongoing data transfer (e.g., continuous DMS CDC replication at scale, or large recurring DataSync transfers).
- **VPC Peering**: Direct private network connection between two VPCs (same or different accounts/regions) — no transitive routing (peering A↔B and B↔C does not let A reach C).
- **AWS Transit Gateway**: Hub-and-spoke connectivity across many VPCs/VPNs/Direct Connect connections — used when peering would otherwise require an unmanageable number of point-to-point connections.

## 6. Common Exam Scenarios & Gotchas

1. **Glue job/EMR/Lambda in a private subnet needs to read/write S3 without a NAT Gateway** → **S3 Gateway Endpoint**.
2. **Private-subnet resource needs to call Secrets Manager/KMS/Kinesis privately** → **Interface Endpoint (PrivateLink)** for that service.
3. **Reduce cost of routing AWS-service traffic through a NAT Gateway** → replace with the appropriate **VPC Endpoint** (Gateway for S3/DynamoDB, Interface for others).
4. **Block a specific IP range at the subnet level regardless of individual security group rules** → **Network ACL** with an explicit Deny rule.
5. **RDS/Redshift/MSK should not be reachable from the public internet** → place in **private subnets**, restrict access via **security groups**.
6. **High-volume, low-latency, ongoing on-prem-to-AWS data replication (e.g., continuous CDC)** → **Direct Connect** over VPN.
7. **Connect many VPCs across an organization without a full peering mesh** → **Transit Gateway**.
8. **A allowed to reach B, B allowed to reach C — does A reach C via VPC Peering?** → **No**, peering is **not transitive**.

## 7. Quick Reference Cheat Sheet

- Security Group = **stateful**, instance-level, allow-only
- Network ACL = **stateless**, subnet-level, allow **and** deny
- Gateway Endpoint = **free**, S3 & DynamoDB only
- Interface Endpoint (PrivateLink) = **paid**, most other AWS services
- NAT Gateway = **outbound internet** access for private subnets (not for AWS-service traffic — use endpoints instead)
- VPC Peering = **not transitive**
- Transit Gateway = **hub-and-spoke** for many VPCs
- Direct Connect = **dedicated, high-bandwidth, low-latency** on-prem link; VPN = **cheaper, internet-based, encrypted tunnel**

---

*Part of the AWS Certified Data Engineer - Associate Exam Study Materials repository.*
