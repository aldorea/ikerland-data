---
phase: 02-data-pipeline-storage-alarm-notifications
plan: "02"
subsystem: storage-layer
tags: [timestream, dynamodb, aurora, storage, vpc, time-series, relational]
dependency_graph:
  requires:
    - 01-security-foundation.md (VPC topology, data subnets, VPC endpoints)
  provides:
    - docs/architecture/05-storage-layer.md (three-tier storage documentation)
  affects:
    - Phase 3 ETL (Timestream export, DynamoDB TTL patterns)
    - Phase 4 API (RDS Proxy, Aurora credential flow)
tech_stack:
  added:
    - Amazon Timestream for LiveAnalytics (hot time-series, memory/magnetic tiering)
    - Amazon DynamoDB On-Demand (operational state, TTL cleanup, alarm dedup)
    - Amazon Aurora PostgreSQL Serverless v2 (relational data, RBAC, alert rules)
    - AWS RDS Proxy (Lambda connection pooling for Aurora)
    - AWS Secrets Manager (Aurora credential rotation)
  patterns:
    - DEDUP#{thingName}#{alarmType} composite key for atomic alarm deduplication
    - DynamoDB TTL for zero-cost transient data cleanup (dedup 15min, cmd-queue 24h, latest-value 1h)
    - Batch WriteRecords to Timestream per Lambda invocation (amortizes API overhead)
    - RDS Proxy mandatory pattern for Lambda → Aurora (prevents connection exhaustion)
key_files:
  created:
    - docs/architecture/05-storage-layer.md
  modified: []
decisions:
  - "Timestream selected for hot time-series: 1000x faster range queries, no per-query cost on memory tier, auto-tiering to magnetic"
  - "DynamoDB dedup pattern: DEDUP#{thingName}#{alarmType} PK with TTL 900s — atomic PutItem with attribute_not_exists condition"
  - "Aurora Serverless v2 minimum 0.5 ACU (~$43/month idle) — not zero — documented honestly in comparison table"
  - "RDS Proxy required (not optional) for Lambda → Aurora — connection exhaustion is #1 production failure mode"
  - "All three storage services in data subnets: Timestream via Interface endpoint (~$7/AZ), DynamoDB via Gateway endpoint (free), Aurora via sg-aurora security group"
metrics:
  duration: "2 minutes"
  completed_date: "2026-03-27"
  tasks_completed: 1
  tasks_total: 1
  files_created: 1
  files_modified: 0
requirements_addressed:
  - STOR-01 (Timestream hot time-series)
  - STOR-02 (DynamoDB operational state)
  - STOR-03 (Aurora Serverless v2 relational)
  - STOR-04 (storage comparison tables)
---

# Phase 02 Plan 02: Storage Layer — Summary

**One-liner:** Three-tier storage with Timestream (memory/magnetic tiering), DynamoDB (5-table design with DEDUP# dedup pattern), and Aurora Serverless v2 (RDS Proxy mandatory, honest 0.5 ACU idle cost), all placed in VPC data subnets with no public endpoints.

## Tasks Completed

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | Three-tier storage architecture with Timestream, DynamoDB, and Aurora documentation | `6dd7230` | docs/architecture/05-storage-layer.md |

## What Was Built

`docs/architecture/05-storage-layer.md` provides complete documentation for the three-tier storage architecture:

**Amazon Timestream for LiveAnalytics (STOR-01)**
- Memory store (24h, no per-query cost) and magnetic store (365 days, $0.01/GB scanned)
- Database `iot-telemetry`, table `device-metrics` with thingName/deviceType/location dimensions
- Write batching strategy: all records from a Lambda invocation in one `WriteRecords` API call
- Interface VPC endpoint required (~$7/month per AZ)

**Amazon DynamoDB (STOR-02)**
- Five table roles: device-metadata, device-latest (TTL 1h), command-queue (TTL 24h), alarm-dedup (TTL 15min), config-state
- `DEDUP#{thingName}#{alarmType}` composite PK for atomic race-condition-free alarm deduplication
- On-Demand billing with Gateway VPC endpoint (free)
- TTL provides zero-cost cleanup for three transient table types

**Amazon Aurora PostgreSQL Serverless v2 (STOR-03)**
- Stores users, roles (RBAC), alert rule configurations, device groups
- Minimum 0.5 ACU idle cost honestly stated: ~$43/month (not zero-cost)
- RDS Proxy documented as required (not optional) for Lambda connection pooling
- Secrets Manager integration for automatic credential rotation

**VPC Placement Diagram**
- Mermaid flowchart showing Lambda functions (private subnets) → all three storage services (data subnets)
- Explicit VPC endpoint labels (Interface vs Gateway) and security group references

**Time-Series Comparison Table (STOR-04)**
- 4 alternatives: Timestream (recommended), DynamoDB time-sorted SK, InfluxDB on EC2, RDS PostgreSQL
- Pros/cons/cost profile/recommendation for each

**Relational Comparison Table (STOR-04)**
- 4 alternatives: Aurora Serverless v2 (recommended), RDS Provisioned, DynamoDB fully NoSQL, Aurora Provisioned
- Pros/cons/cost profile/recommendation for each; Aurora minimum idle cost explicitly compared

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all sections are fully documented. No placeholder data, no "coming soon" markers.

## Self-Check: PASSED

- `docs/architecture/05-storage-layer.md` — FOUND
- Commit `6dd7230` — FOUND
