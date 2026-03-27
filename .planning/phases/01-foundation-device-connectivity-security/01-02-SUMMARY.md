---
phase: 01-foundation-device-connectivity-security
plan: 02
subsystem: device-connectivity-ingestion
tags: [iot-core, kinesis, mqtt, rules-engine, firehose, s3, dlq]
dependency_graph:
  requires: []
  provides:
    - docs/architecture/02-device-connectivity-ingestion.md
  affects:
    - Phase 2 processing pipeline (consumes Kinesis Data Streams)
    - Phase 3 Data Lake ETL (S3 bronze zone delivery by Firehose)
tech_stack:
  added: []
  patterns:
    - IoT Core MQTT entry point (port 8883, mutual TLS, Basic Ingest for cost optimization)
    - Topic namespace: devices/{thingName}/{messageType}
    - IoT Rules Engine SQL routing with topic(2) function and errorAction DLQ
    - Kinesis Data Streams as telemetry buffer (On-Demand, thingName partition key)
    - Kinesis Data Firehose for S3 delivery with dynamic time-based partitioning
key_files:
  created:
    - docs/architecture/02-device-connectivity-ingestion.md
  modified: []
decisions:
  - "Kinesis over direct Lambda for telemetry — batch processing amortizes cost, provides 24h replay, and decouples ingestion from processing rate"
  - "Basic Ingest documented for high-volume scenarios — bypasses pub/sub broker, ~50% cost reduction for one-way telemetry"
  - "errorAction on all IoT Rules → SQS DLQ (iot-rules-dlq) — no silent message loss paths"
  - "Dynamic Firehose partitioning by year=/month=/day=/ enables Athena partition pruning"
metrics:
  duration_minutes: 1
  completed_date: "2026-03-27"
  tasks_completed: 2
  tasks_total: 2
  files_created: 1
  files_modified: 0
---

# Phase 01 Plan 02: Device Connectivity and Ingestion Layer Summary

**One-liner:** IoT Core MQTT/HTTPS entry point documented with topic namespace, Rules Engine SQL routing (3 rules, all with DLQ errorAction), and Kinesis Firehose ingestion pipeline to S3 bronze zone.

## What Was Built

Documentation file `docs/architecture/02-device-connectivity-ingestion.md` covering the complete device-to-cloud connectivity and ingestion layer:

- IoT Core entry point with protocol comparison table (MQTT 8883 mTLS, MQTT-WS 443, HTTPS 443)
- Basic Ingest cost optimization pattern (~50% cost reduction for one-way telemetry)
- 7-row topic namespace table with exact patterns, directions, payload examples, and consumers
- Mermaid flowchart LR showing device → IoT Core → Rules Engine fan-out → Kinesis pipeline
- 3-rule Rules Engine inventory table with full SQL statements, targets, and errorAction DLQ references
- topic(2) function explanation and partition key strategy note
- Kinesis vs Lambda vs SQS comparison table with clear recommendation for telemetry hot path
- Kinesis Data Streams On-Demand configuration details
- Kinesis Data Firehose S3 delivery configuration with dynamic time-based partitioning

## Requirements Addressed

| ID | Description | Status |
|----|-------------|--------|
| INGT-01 | IoT Core as MQTT/HTTPS entry point (protocol table, port 8883) | Complete |
| INGT-02 | Rules Engine routing with SQL expressions (3-rule inventory table) | Complete |
| INGT-03 | Kinesis Firehose as ingestion buffer (comparison table + configuration) | Complete |
| INGT-04 | Topic namespace design (7-row topic table with patterns and payload examples) | Complete |

## Commits

| Task | Name | Commit | Files |
|------|------|--------|-------|
| 1 | IoT Core entry point, topic namespace, ingestion data flow diagram | 42e5041 | docs/architecture/02-device-connectivity-ingestion.md (created) |
| 2 | Rules Engine routing table and Kinesis ingestion buffer | (included in 42e5041) | docs/architecture/02-device-connectivity-ingestion.md (same file) |

> **Note:** Task 2 content was written as part of the full document creation in Task 1 commit. Both tasks' acceptance criteria are verified independently and passed.

## Decisions Made

1. **Kinesis over direct Lambda for telemetry** — Comparison table explicitly documents three approaches (IoT Rules → Lambda direct, SQS → Lambda, Kinesis → Lambda) with cost, replay, ordering, and backpressure analysis. Kinesis wins on all axes for high-volume telemetry.

2. **Basic Ingest documented** — Cost optimization note included per plan spec. Reserved topics bypass the pub/sub broker, reducing IoT Core messaging costs by ~50% for one-way telemetry flows.

3. **errorAction on every rule** — TelemetryRule, AlarmRule, and ConfigRule all route failures to `iot-rules-dlq` SQS queue. This prevents silent message loss and provides a replayable audit trail of routing failures.

4. **Dynamic Firehose partitioning** — `year=/month=/day=/` prefix pattern using JQ expression on the `ts` field enables efficient partition pruning in Athena queries without full table scans.

## Deviations from Plan

None — plan executed exactly as written. Task 1 and Task 2 content were written in a single file creation (no append needed since the full document was designed from the start), but all acceptance criteria for both tasks pass independently.

## Known Stubs

None — this is a documentation-only plan. No data wiring, UI components, or code stubs exist.

## Self-Check: PASSED

- [x] `docs/architecture/02-device-connectivity-ingestion.md` exists and contains all required content
- [x] Commit 42e5041 exists in git history
- [x] All Task 1 acceptance criteria verified (flowchart LR, topic patterns, port 8883, Basic Ingest, shadow topics, DLQ)
- [x] All Task 2 acceptance criteria verified (TelemetryRule/AlarmRule/ConfigRule, SQL statements, errorAction x3, iot-rules-dlq, topic(2), Approach table, Firehose config, dynamic partitioning)
