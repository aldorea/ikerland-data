---
phase: 01-foundation-device-connectivity-security
plan: "01"
subsystem: security-foundation
tags: [vpc, iam, kms, tls, waf, security, networking, documentation]
dependency_graph:
  requires: []
  provides:
    - docs/architecture/01-security-foundation.md
  affects:
    - All subsequent phases referencing VPC topology and security controls
tech_stack:
  added: []
  patterns:
    - Mermaid flowchart TD with nested subgraph blocks for VPC tier visualization
    - Explicit Gateway vs Interface VPC endpoint type labeling
    - IAM roles inventory table with Role Name / Assumed By / Permissions / Scope columns
    - KMS CMK per-service encryption table
key_files:
  created:
    - docs/architecture/01-security-foundation.md
  modified: []
decisions:
  - "Single NAT Gateway for cost optimization with note that production HA uses one per AZ (D-03)"
  - "Gateway endpoints (S3, DynamoDB) are free; Interface endpoints used for Timestream, IoT Core, Secrets Manager, SQS, Kinesis (~$7/month per AZ each)"
  - "rt-data route table has no 0.0.0.0/0 entry — data tier is fully isolated from internet"
  - "One CMK per service category (storage, streaming, audit) recommended for KMS"
  - "WAF WebACLs on CloudFront and API Gateway; IoT Core MQTT excluded (X.509 mTLS provides equivalent protection)"
metrics:
  duration_minutes: 2
  completed_date: "2026-03-27"
  tasks_completed: 2
  files_created: 1
  files_modified: 0
---

# Phase 01 Plan 01: Security Foundation Summary

**One-liner:** 3-tier VPC topology with Mermaid diagram, 10-role IAM inventory, KMS CMK per-service encryption, TLS 1.2+ enforcement points, and dual WAF WebACL placement documented for evaluator verification.

## What Was Built

The file `docs/architecture/01-security-foundation.md` was created as a standalone, evaluator-ready architecture documentation section covering all six SEC requirements.

### Task 1: VPC Topology, Endpoints, Security Groups, Route Tables

- **VPC topology diagram:** Mermaid `flowchart TD` with 10 nested `subgraph` blocks — VPC container, Public Tier with 2 AZ subgraphs, Private Tier with 2 AZ subgraphs, Data Tier with 2 AZ subgraphs
- **Subnet inventory table:** All 6 subnets with AZ, CIDR (`10.0.0.0/16` base), tier, and services hosted
- **VPC endpoints table:** 7 services explicitly typed as Gateway (S3, DynamoDB — free) or Interface (Timestream, IoT Core, Secrets Manager, SQS, Kinesis — ~$7/month per AZ)
- **Security groups table:** 4 groups (sg-lambda, sg-aurora, sg-alb, sg-vpce) with precise inbound/outbound rules
- **Route tables:** 3 route tables; `rt-data` explicitly has no `0.0.0.0/0` entry confirming database internet isolation

### Task 2: IAM Roles, KMS, TLS, WAF

- **IAM roles inventory:** 10 roles covering all IoT Rules Engine actions (4 roles), Lambda handlers (3 roles), Glue ETL, Firehose delivery, and Cognito auth — each with specific ARN scope
- **KMS encryption table:** 6 services (S3, DynamoDB, Timestream, Aurora, Kinesis, CloudTrail) each using Customer-managed CMK with description of what is protected
- **TLS enforcement:** 5 bullet points covering IoT Core MQTT port 8883 (mTLS), API Gateway, CloudFront (`TLSv1.2_2021`), Aurora (`rds.force_ssl=1`), and VPC endpoint traffic
- **WAF placement:** 2 WebACLs — `waf-cloudfront` (CloudFront + AWSManagedRulesCommonRuleSet + KnownBadInputs + rate limit 1000/5min) and `waf-api` (API Gateway + CommonRuleSet + SQLiRuleSet + rate limit 500/5min)

## Requirements Addressed

| ID | Requirement | Status |
|----|-------------|--------|
| SEC-01 | VPC topology with private subnets for all databases | Complete — 3-tier diagram + subnet table + rt-data has no internet route |
| SEC-02 | VPC endpoints with Gateway/Interface type labels | Complete — 7 endpoints explicitly typed in table and diagram nodes |
| SEC-03 | IAM least-privilege roles for all service-to-service communication | Complete — 10-role inventory table with scoped ARNs |
| SEC-04 | KMS encryption at rest for all data stores | Complete — 6-service CMK table |
| SEC-05 | TLS 1.2+ in transit | Complete — 5 enforcement points including IoT Core port 8883 mTLS |
| SEC-06 | WAF on CloudFront and API Gateway | Complete — 2 WebACLs with managed rule groups named |

## Commits

| Task | Commit | Files |
|------|--------|-------|
| Tasks 1 + 2 | d07eb2b | docs/architecture/01-security-foundation.md (created, 185 lines) |

## Deviations from Plan

None — plan executed exactly as written. Both tasks were completed in a single file write since the plan output was a single document. The file was committed after both tasks were complete as the document needed to be coherent as a whole.

## Known Stubs

None — all content is fully specified with concrete values (CIDR blocks, role names, ARNs, rule group names, port numbers). No placeholder text, no "coming soon" sections.

## Self-Check: PASSED
