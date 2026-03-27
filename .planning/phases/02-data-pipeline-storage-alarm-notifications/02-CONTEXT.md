# Phase 2: Data Pipeline, Storage & Alarm Notifications - Context

**Gathered:** 2026-03-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Document how telemetry flows from IoT Core through Kinesis into Timestream/DynamoDB (hot processing path), how relational data is stored in Aurora Serverless v2, and how alarm events are detected via IoT Rules Engine threshold evaluation, deduplicated, and delivered through SNS fan-out to email (SES) with EventBridge extensibility for future channels. Output is architecture documentation with Mermaid diagrams and technology comparison tables.

</domain>

<decisions>
## Implementation Decisions

### Processing Pipeline
- **D-01:** Document Kinesis Data Streams batch processing as the primary telemetry hot path. Include batch window configuration (100-500 records per batch), cold start amortization rationale, and 24h replay capability.
- **D-02:** Include a concrete JSON transformation example showing raw device telemetry → normalized format with enrichment from DynamoDB device metadata lookup.
- **D-03:** Comparison table: Kinesis batch processing vs direct per-message Lambda invocation vs SQS-based processing — with throughput, cost, replay, and operational complexity as evaluation axes.
- **D-04:** SQS dead-letter queues on ALL Lambda consumers (both telemetry processor and alarm evaluator). Document maxReceiveCount, retention period, and CloudWatch alarm on DLQ depth. No silent data loss paths.

### Storage Tiers
- **D-05:** Three-tier storage architecture:
  - **Timestream** — hot time-series telemetry (memory tier: hours, magnetic tier: months). Purpose-built for time-range queries and aggregations.
  - **DynamoDB** — device metadata, latest-value cache, command queue state, alarm dedup records. On-Demand billing, TTL for transient data.
  - **Aurora Serverless v2** — relational data: users, roles, alert rule configurations, device groups. Scales to 0.5 ACU when idle.
- **D-06:** All three storage services accessed exclusively from VPC private/data subnets (inheriting Phase 1 D-01 VPC topology). No public endpoints.
- **D-07:** Storage comparison table with 4 alternatives per decision point:
  - Time-series: Timestream vs DynamoDB (time-sorted SK) vs InfluxDB on EC2 vs RDS time-series
  - Relational: Aurora Serverless v2 vs RDS Provisioned vs DynamoDB (fully NoSQL) vs Aurora Provisioned
  - Each with pros, cons, cost profile, and justified recommendation.

### Alarm Detection & Notification
- **D-08:** IoT Rules Engine SQL for threshold breach detection as the primary alarm trigger pattern. Example: `SELECT * FROM 'devices/+/alarm' WHERE value > threshold`.
- **D-09:** Lambda alarm evaluator receives alarm events, checks DynamoDB dedup table for recent alerts within a configurable time window (default 15 min), and only triggers notification if no duplicate found. DynamoDB TTL auto-cleans expired dedup records.
- **D-10:** SNS Standard topic as the fan-out hub. Primary subscription: Lambda → SES for email notifications. SNS chosen over direct SES invocation for multi-channel extensibility.
- **D-11:** EventBridge documented as the extensibility layer for future notification channels (SMS, webhook, Slack, PagerDuty). Show the EventBridge rule pattern but do not implement each channel in detail — this is an architecture extensibility point.
- **D-12:** Rate-of-change and multi-signal correlation noted as v2 alarm enhancements achievable via EventBridge rules with pattern matching. Phase 2 focuses on threshold breach as the core pattern.

### Documentation Format
- **D-13:** Continue Phase 1 pattern: single markdown sections per architectural layer, Mermaid diagrams for all data flows, inline comparison tables with alternatives/pros/cons/recommendation.
- **D-14:** Include a data flow Mermaid diagram showing the complete hot path: IoT Core → Kinesis Data Streams → Lambda processor → Timestream + DynamoDB, with DLQ branches.
- **D-15:** Include a separate alarm pipeline Mermaid diagram: IoT Core → Lambda evaluator → DynamoDB dedup check → SNS → SES email, with EventBridge extensibility branch.

### Claude's Discretion
- Exact Kinesis batch window size (100-500 range documented, specific number is implementation detail)
- Mermaid diagram layout and styling within established conventions
- Level of detail in Lambda transformation code examples (pseudo-code vs detailed)
- Whether to show Timestream memory/magnetic retention periods as specific values or configurable parameters
- EventBridge rule pattern syntax depth

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Project scope, constraints, evaluation criteria
- `.planning/REQUIREMENTS.md` — Full requirement list; Phase 2 maps: PROC-01..03, STOR-01..04, ALRM-01..04

### Prior Phase Output
- `docs/architecture/01-security-foundation.md` — VPC topology, IAM roles, KMS, WAF (Phase 2 storage inherits this)
- `docs/architecture/02-device-connectivity-ingestion.md` — IoT Core, topic namespace, Rules Engine routing, Kinesis setup (Phase 2 continues from here)
- `docs/architecture/03-device-management.md` — Device Shadow, Fleet Provisioning (referenced by alarm pipeline)

### Phase 1 Context
- `.planning/phases/01-foundation-device-connectivity-security/01-CONTEXT.md` — Prior decisions D-01 through D-13 (VPC, provisioning, topic namespace, doc format)

### Research
- `.planning/research/STACK.md` — AWS service recommendations with alternatives and rationale
- `.planning/research/ARCHITECTURE.md` — Component boundaries, data flows, architectural patterns
- `.planning/research/PITFALLS.md` — Critical mistakes to avoid (relevant: Timestream pricing gotchas, Lambda DLQ patterns)
- `.planning/research/SUMMARY.md` — Synthesized research findings

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `docs/architecture/01-security-foundation.md` — VPC topology diagram and subnet inventory (Phase 2 storage placement references this directly)
- `docs/architecture/02-device-connectivity-ingestion.md` — IoT Rules Engine routing table and Kinesis setup (Phase 2 processing pipeline continues from this endpoint)
- Established Mermaid diagram style and comparison table format from Phase 1 docs

### Established Patterns
- Architecture doc sections follow: narrative introduction → table/inventory → Mermaid diagram → comparison table (if applicable) → design notes
- Comparison tables use: Alternative | Pros | Cons | Recommendation columns
- Decision references use D-XX numbering linked to CONTEXT.md

### Integration Points
- Phase 2 processing pipeline starts where Phase 1 Kinesis Data Streams ends (IoT Rules Engine → Kinesis is already documented)
- Storage services (Timestream, DynamoDB, Aurora) must be placed in data subnets per Phase 1 VPC topology
- Alarm pipeline originates from `devices/{thingName}/alarm` topic already defined in Phase 1 topic namespace
- DLQ pattern from Phase 1 (D-10 errorAction → SQS) extends to Lambda consumer DLQs in Phase 2

</code_context>

<specifics>
## Specific Ideas

- The Kinesis vs per-message Lambda comparison table is explicitly called out in Success Criteria #1 — must be convincing with concrete numbers (throughput, cost at scale)
- Storage comparison (Success Criteria #3) must cover at least three alternatives — we're doing four for depth
- The alarm pipeline sequence (Success Criteria #4) must be traceable end-to-end: Rules Engine SQL → Lambda evaluator → dedup check → SNS → SES
- DLQ coverage (Success Criteria #5) must be exhaustive — no Lambda consumer without a DLQ
- Evaluators value the deduplication strategy (ALRM-04) as a signal of production-readiness thinking

</specifics>

<deferred>
## Deferred Ideas

- Rate-of-change alarm detection (multi-reading trend analysis) — v2 enhancement via EventBridge
- Multi-signal correlation alarms (e.g., temperature + humidity combined threshold) — v2 via EventBridge pattern matching
- Step Functions for multi-step alarm escalation workflows (warn → escalate → page on-call) — Phase 4 or v2
- Amazon Managed Grafana as real-time monitoring dashboard connected to Timestream — Phase 4 consideration
- DynamoDB DAX caching layer for high-frequency device metadata lookups — v2 optimization

</deferred>

---

*Phase: 02-data-pipeline-storage-alarm-notifications*
*Context gathered: 2026-03-27*
