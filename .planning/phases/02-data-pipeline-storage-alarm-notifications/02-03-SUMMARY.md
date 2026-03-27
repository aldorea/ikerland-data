---
phase: 02-data-pipeline-storage-alarm-notifications
plan: "03"
subsystem: alarm-notifications
tags: [alarm, iot-rules-engine, lambda, dynamodb, sns, ses, eventbridge, deduplication, dlq]
dependency_graph:
  requires:
    - docs/architecture/02-device-connectivity-ingestion.md  # AlarmRule SQL and topic namespace
    - .planning/phases/02-data-pipeline-storage-alarm-notifications/02-CONTEXT.md  # D-08 through D-12, D-15
  provides:
    - docs/architecture/06-alarm-notifications.md
  affects:
    - docs/architecture/04-data-pipeline-processing.md  # references same DLQ pattern (complementary)
tech_stack:
  added: []
  patterns:
    - DynamoDB conditional write deduplication with TTL (attribute_not_exists pattern)
    - SNS fan-out decoupling alarm evaluator from delivery channels
    - EventBridge custom bus as extensibility hook for future notification channels
    - Lambda function DeadLetterConfig for async-invoked Lambdas (vs ESM DestinationConfig for stream consumers)
key_files:
  created:
    - docs/architecture/06-alarm-notifications.md
  modified: []
decisions:
  - "D-08: IoT Rules Engine WHERE clause as coarse pre-filter before Lambda evaluator — prevents unnecessary invocations at scale"
  - "D-09: DynamoDB conditional write (attribute_not_exists) with 15-min TTL for per-device per-alarm-type deduplication"
  - "D-10: SNS Standard topic alarm-notifications as fan-out hub — decouples evaluator from delivery channels"
  - "D-11: EventBridge iot-alarm-bus as extensibility layer for future channels — zero alarm evaluator code changes needed"
  - "D-12: Rate-of-change and multi-signal correlation deferred to v2 — EventBridge hook is already in place"
  - "D-15: Separate alarm pipeline Mermaid flowchart with dashed lines for failure and future-extension paths"
metrics:
  duration: 1m
  completed_date: "2026-03-27"
  tasks_completed: 1
  tasks_total: 1
  files_created: 1
  files_modified: 0
---

# Phase 2 Plan 03: Alarm Notifications — Summary

**One-liner:** IoT Rules Engine SQL threshold pre-filter + Lambda evaluator with DynamoDB TTL dedup (attribute_not_exists, 15-min window) + SNS fan-out to SES email + EventBridge iot-alarm-bus extensibility hook for future channels.

## What Was Built

`docs/architecture/06-alarm-notifications.md` — Complete alarm detection and notification pipeline documentation covering:

1. **Alarm pipeline component inventory** — 7-component table (threshold pre-filter, alarm evaluator, dedup table, SNS hub, SES email, EventBridge bus, DLQ)
2. **IoT Rules Engine threshold detection** — AlarmRule SQL with `WHERE value > 80` coarse pre-filter, anti-pattern warning for missing WHERE clause at 1,000-device scale
3. **Lambda alarm evaluator 8-step flow** — from async invocation through per-device Aurora threshold check, DynamoDB conditional write dedup, SNS publish, and EventBridge PutEvents
4. **Deduplication strategy** — DynamoDB composite key `DEDUP#{thingName}#{alarmType}`, 15-minute TTL, `attribute_not_exists(PK)` condition, Python pseudo-code with CloudWatch `AlarmsDeduplicated` metric on suppressed alarms
5. **SNS fan-out and SES email** — `alarm-notifications` Standard topic, `ses-email-sender` Lambda subscription, `SendTemplatedEmail`, bounce/complaint tracking, SES pricing
6. **EventBridge extensibility** — `iot-alarm-bus` custom bus, `PutEvents` code example with `Source: iot.alarm-evaluator`, future channel table (SMS, webhook, Slack/PagerDuty via API Destination, EventBridge Pipes)
7. **Alarm pipeline Mermaid diagram** — `flowchart LR` from IoT Device through IoT Core, AlarmRule, Lambda evaluator, DynamoDB dedup decision, SNS, SES, EventBridge, with dashed lines for DLQ and future channels
8. **DLQ configuration** — explains why alarm evaluator uses Lambda function `DeadLetterConfig` (async invocation), not ESM `DestinationConfig` (stream consumer); contrast table with telemetry processor

## Task Commits

| Task | Name | Commit | Files |
|---|---|---|---|
| 1 | Alarm detection pipeline documentation | 77d90d3 | docs/architecture/06-alarm-notifications.md |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — this is a documentation-only deliverable. No data-wiring or UI components.

## Self-Check: PASSED

- [x] `docs/architecture/06-alarm-notifications.md` exists and is non-empty
- [x] Mermaid `flowchart` diagram present with Device, IoT Core, AlarmRule, Lambda, DynamoDB, SNS, SES, EventBridge, SQS DLQ
- [x] `WHERE value > 80` SQL present
- [x] `DEDUP#{thingName}#{alarmType}` composite key documented
- [x] `attribute_not_exists(PK)` conditional expression present
- [x] 15-minute TTL (900 seconds) documented
- [x] `alarm-notifications` SNS topic name present
- [x] `iot-alarm-bus` EventBridge custom bus present
- [x] `SendTemplatedEmail` SES method documented
- [x] `sqs-alarm-dlq` DLQ name present
- [x] `AlarmsDeduplicated` CloudWatch metric present
- [x] EventBridge PutEvents code with `"Source": "iot.alarm-evaluator"` present
- [x] 8-step alarm evaluator flow (numbered list) present
- [x] v2 mention for rate-of-change and multi-signal correlation present (D-12)
- [x] Anti-pattern warning for missing WHERE clause present
- [x] Commit 77d90d3 confirmed in git log
