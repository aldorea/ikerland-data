---
phase: 02-data-pipeline-storage-alarm-notifications
plan: "01"
subsystem: data-pipeline
tags: [kinesis, lambda, timestream, dynamodb, sqs-dlq, iot-processing, arm64, graviton2]

requires:
  - phase: 01-foundation-device-connectivity-security
    provides: Kinesis Data Streams endpoint, VPC topology (private/data subnets), IoT Rules Engine routing table

provides:
  - Complete telemetry hot-path processing pipeline documentation (04-data-pipeline-processing.md)
  - Kinesis ESM batch consumer configuration with BisectBatchOnFunctionError strategy
  - JSON transformation chain: raw device payload → DynamoDB enrichment → Timestream WriteRecords + DynamoDB latest-value cache
  - Processing approach comparison table: per-message Lambda vs SQS vs Kinesis batch
  - SQS DLQ configuration for both Lambda consumers with CloudWatch alarms
  - ESM vs function-level DLQ distinction documented

affects:
  - 02-02 (storage layer — Timestream/DynamoDB write patterns established here)
  - 02-03 (alarm notifications — alarm-evaluator DLQ pattern established here)

tech-stack:
  added: []
  patterns:
    - "Kinesis ESM batch processing: BatchSize=200, BisectBatchOnFunctionError, MaximumRetryAttempts=3, DestinationConfig.OnFailure → SQS DLQ"
    - "DynamoDB BatchGetItem for enrichment: one call per Kinesis batch (not per record) to minimize read unit consumption"
    - "Timestream WriteRecords: one API call per batch (not per record) to minimize write unit cost"
    - "DLQ placement rule: ESM consumers use DestinationConfig.OnFailure; async-invoked Lambdas use function-level DeadLetterConfig"

key-files:
  created:
    - docs/architecture/04-data-pipeline-processing.md
  modified: []

key-decisions:
  - "Kinesis batch processing selected over per-message Lambda (10-20x fewer invocations) and SQS (no replay capability) for telemetry hot path"
  - "BatchSize=200 with BisectBatchOnFunctionError isolates poison-pill records without discarding entire batches"
  - "arm64 (Graviton2) runtime for telemetry-processor Lambda: 20% cost reduction at equivalent performance"
  - "DLQ distinction: ESM DestinationConfig.OnFailure for Kinesis consumers; function-level DeadLetterConfig for async-invoked alarm-evaluator"

patterns-established:
  - "Pattern: Kinesis ESM configuration table (Parameter | Value | Rationale format)"
  - "Pattern: JSON transformation chain shown as three labeled steps with code blocks"
  - "Pattern: Processing approach comparison with 5 axes (Throughput, Cost, Replay, Ordering, Complexity)"

requirements-completed: [PROC-01, PROC-02, PROC-03]

duration: 7min
completed: 2026-03-27
---

# Phase 02 Plan 01: Data Pipeline Processing Summary

**Kinesis batch Lambda consumer (BatchSize=200, BisectBatchOnFunctionError, Graviton2/arm64) with JSON transformation chain, Timestream/DynamoDB writes, and SQS DLQ coverage for both Lambda consumers**

## Performance

- **Duration:** ~7 min
- **Started:** 2026-03-27T16:33:58Z
- **Completed:** 2026-03-27T16:41:00Z
- **Tasks:** 1 of 1
- **Files modified:** 1

## Accomplishments

- Created `docs/architecture/04-data-pipeline-processing.md` with complete hot-path processing pipeline documentation
- Documented Kinesis ESM configuration (BatchSize=200, BisectBatchOnFunctionError, MaximumRetryAttempts=3, Graviton2/arm64) per D-01
- Documented complete JSON transformation chain — raw device payload → DynamoDB device-metadata enrichment → Timestream WriteRecords format + DynamoDB latest-value cache write — per D-02
- Created 3-row processing approach comparison table (per-message Lambda, SQS, Kinesis batch) with 5 evaluation axes and "Recommended" marker per D-03
- Documented SQS DLQ configuration for both Lambda consumers (`sqs-telemetry-dlq`, `sqs-alarm-dlq`) with maxReceiveCount=3, 14-day retention, and CloudWatch alarms per D-04
- Explicitly documented the ESM vs function-level DLQ distinction — critical evaluator checkpoint per PROC-02

## Task Commits

1. **Task 1: Kinesis batch processing pipeline with comparison table and data flow diagram** - `6a10efc` (feat)

**Plan metadata:** (pending final commit)

## Files Created/Modified

- `docs/architecture/04-data-pipeline-processing.md` — Complete telemetry hot-path processing pipeline: Lambda ESM config table, JSON transformation chain with code examples, flowchart LR Mermaid diagram (IoT Core → Kinesis → Lambda → Timestream/DynamoDB with DLQ), 3-row comparison table, DLQ configuration table for both Lambda consumers, anti-patterns section

## Decisions Made

- Kinesis batch processing confirmed as recommended approach — documented with concrete cost rationale (10-20x fewer invocations vs per-message Lambda) and replay capability advantage over SQS
- BisectBatchOnFunctionError strategy documented to explain poison-pill record isolation without batch discard
- Critical architectural distinction documented: Kinesis ESM consumers require `DestinationConfig.OnFailure` on the ESM resource, NOT `DeadLetterConfig` on the Lambda function — the latter only applies to async invocations

## Deviations from Plan

None — plan executed exactly as written. All sections specified in the plan action were implemented in the prescribed order with content sourced from 02-RESEARCH.md and 02-CONTEXT.md.

## Issues Encountered

None.

## User Setup Required

None — no external service configuration required. This is a documentation-only deliverable.

## Next Phase Readiness

- `04-data-pipeline-processing.md` is complete and provides the Timestream/DynamoDB write patterns that Plan 02 (storage layer) will reference
- Kinesis endpoint documented in `02-device-connectivity-ingestion.md` (Phase 1) is now connected forward to the Lambda batch consumer configuration
- Both Lambda consumer DLQs are documented — Plan 03 (alarm notifications) can reference `sqs-alarm-dlq` for the alarm-evaluator DLQ coverage

---

*Phase: 02-data-pipeline-storage-alarm-notifications*
*Completed: 2026-03-27*
