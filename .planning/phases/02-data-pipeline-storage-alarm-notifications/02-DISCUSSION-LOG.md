# Phase 2: Data Pipeline, Storage & Alarm Notifications - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-27
**Phase:** 02-data-pipeline-storage-alarm-notifications
**Areas discussed:** Processing pipeline depth, Storage comparison scope, Alarm detection complexity, Alarm deduplication strategy, Notification channel breadth
**Mode:** --auto (all selections made by recommended defaults)

---

## Processing Pipeline Documentation Depth

| Option | Description | Selected |
|--------|-------------|----------|
| Batch-focused with transformation examples | Document Kinesis batch config, show JSON transformation example, DLQ pattern | [auto] |
| High-level flow only | Describe the pipeline conceptually without concrete examples | |
| Implementation-heavy | Include Lambda code snippets, IAM policies, CloudWatch alarms | |

**User's choice:** [auto] Batch-focused with transformation examples (recommended default)
**Notes:** Matches success criteria #1 requiring convincing comparison table. Transformation examples demonstrate practical understanding without crossing into implementation code.

---

## Storage Comparison Scope

| Option | Description | Selected |
|--------|-------------|----------|
| 4 alternatives per decision | Timestream vs DynamoDB vs InfluxDB vs RDS; Aurora vs DynamoDB vs RDS Provisioned vs Aurora Provisioned | [auto] |
| 3 alternatives (minimum) | Meet success criteria minimum | |
| 2 alternatives only | Quick comparison, recommendation + runner-up | |

**User's choice:** [auto] 4 alternatives per decision (recommended default)
**Notes:** Success criteria #3 requires "at least three alternatives." Four shows depth while staying manageable for a design document.

---

## Alarm Detection Complexity

| Option | Description | Selected |
|--------|-------------|----------|
| Threshold breach with extensibility note | IoT Rules Engine SQL for simple thresholds, note EventBridge for future complexity | [auto] |
| Full multi-signal correlation | Document rate-of-change, combined metric thresholds, trend detection | |
| Minimal threshold only | Just the basic threshold breach, no extensibility discussion | |

**User's choice:** [auto] Threshold breach with extensibility note (recommended default)
**Notes:** Keeps Phase 2 focused on the core pattern while showing architectural awareness of future needs. Complex alarm logic deferred to v2.

---

## Alarm Deduplication Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| DynamoDB time-window dedup | Lambda checks DynamoDB for recent alerts within configurable window, TTL auto-cleanup | [auto] |
| Lambda idempotency with Powertools | Use AWS Lambda Powertools idempotency decorator | |
| EventBridge dedup rules | Content-based dedup via EventBridge rule patterns | |

**User's choice:** [auto] DynamoDB time-window dedup (recommended default)
**Notes:** Simplest approach leveraging existing DynamoDB. TTL eliminates maintenance. Configurable window provides operational flexibility. Lambda Powertools is function-level (prevents reprocessing same event), not business-level dedup (prevents same alarm type within time window).

---

## Notification Channel Breadth

| Option | Description | Selected |
|--------|-------------|----------|
| Email (SES) primary + EventBridge extensibility | Full SES email documentation, EventBridge as future channel hook | [auto] |
| Multi-channel detail | Document email, SMS, webhook, Slack each with full implementation detail | |
| Email only | Just SES, no extensibility discussion | |

**User's choice:** [auto] Email (SES) primary + EventBridge extensibility (recommended default)
**Notes:** ALRM-02 requires SNS fan-out for email, ALRM-03 requires EventBridge for extensibility. This covers both requirements without overengineering channels that aren't in scope.

---

## Claude's Discretion

- Kinesis batch window exact size within 100-500 range
- Mermaid diagram layout and styling
- Lambda transformation pseudo-code depth
- Timestream retention period values (specific vs configurable)
- EventBridge rule pattern syntax depth

## Deferred Ideas

- Rate-of-change alarm detection → v2 enhancement
- Multi-signal correlation alarms → v2 via EventBridge
- Step Functions alarm escalation → Phase 4 or v2
- Managed Grafana dashboard → Phase 4
- DynamoDB DAX caching → v2 optimization
