# Phase 2: Data Pipeline, Storage & Alarm Notifications - Research

**Researched:** 2026-03-27
**Domain:** AWS telemetry processing pipeline, multi-tier IoT storage, alarm detection and notification fan-out
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Processing Pipeline
- **D-01:** Document Kinesis Data Streams batch processing as the primary telemetry hot path. Include batch window configuration (100-500 records per batch), cold start amortization rationale, and 24h replay capability.
- **D-02:** Include a concrete JSON transformation example showing raw device telemetry → normalized format with enrichment from DynamoDB device metadata lookup.
- **D-03:** Comparison table: Kinesis batch processing vs direct per-message Lambda invocation vs SQS-based processing — with throughput, cost, replay, and operational complexity as evaluation axes.
- **D-04:** SQS dead-letter queues on ALL Lambda consumers (both telemetry processor and alarm evaluator). Document maxReceiveCount, retention period, and CloudWatch alarm on DLQ depth. No silent data loss paths.

#### Storage Tiers
- **D-05:** Three-tier storage architecture:
  - **Timestream** — hot time-series telemetry (memory tier: hours, magnetic tier: months). Purpose-built for time-range queries and aggregations.
  - **DynamoDB** — device metadata, latest-value cache, command queue state, alarm dedup records. On-Demand billing, TTL for transient data.
  - **Aurora Serverless v2** — relational data: users, roles, alert rule configurations, device groups. Scales to 0.5 ACU when idle.
- **D-06:** All three storage services accessed exclusively from VPC private/data subnets (inheriting Phase 1 D-01 VPC topology). No public endpoints.
- **D-07:** Storage comparison table with 4 alternatives per decision point:
  - Time-series: Timestream vs DynamoDB (time-sorted SK) vs InfluxDB on EC2 vs RDS time-series
  - Relational: Aurora Serverless v2 vs RDS Provisioned vs DynamoDB (fully NoSQL) vs Aurora Provisioned
  - Each with pros, cons, cost profile, and justified recommendation.

#### Alarm Detection & Notification
- **D-08:** IoT Rules Engine SQL for threshold breach detection as the primary alarm trigger pattern. Example: `SELECT * FROM 'devices/+/alarm' WHERE value > threshold`.
- **D-09:** Lambda alarm evaluator receives alarm events, checks DynamoDB dedup table for recent alerts within a configurable time window (default 15 min), and only triggers notification if no duplicate found. DynamoDB TTL auto-cleans expired dedup records.
- **D-10:** SNS Standard topic as the fan-out hub. Primary subscription: Lambda → SES for email notifications. SNS chosen over direct SES invocation for multi-channel extensibility.
- **D-11:** EventBridge documented as the extensibility layer for future notification channels (SMS, webhook, Slack, PagerDuty). Show the EventBridge rule pattern but do not implement each channel in detail — this is an architecture extensibility point.
- **D-12:** Rate-of-change and multi-signal correlation noted as v2 alarm enhancements achievable via EventBridge rules with pattern matching. Phase 2 focuses on threshold breach as the core pattern.

#### Documentation Format
- **D-13:** Continue Phase 1 pattern: single markdown sections per architectural layer, Mermaid diagrams for all data flows, inline comparison tables with alternatives/pros/cons/recommendation.
- **D-14:** Include a data flow Mermaid diagram showing the complete hot path: IoT Core → Kinesis Data Streams → Lambda processor → Timestream + DynamoDB, with DLQ branches.
- **D-15:** Include a separate alarm pipeline Mermaid diagram: IoT Core → Lambda evaluator → DynamoDB dedup check → SNS → SES email, with EventBridge extensibility branch.

### Claude's Discretion
- Exact Kinesis batch window size (100-500 range documented, specific number is implementation detail)
- Mermaid diagram layout and styling within established conventions
- Level of detail in Lambda transformation code examples (pseudo-code vs detailed)
- Whether to show Timestream memory/magnetic retention periods as specific values or configurable parameters
- EventBridge rule pattern syntax depth

### Deferred Ideas (OUT OF SCOPE)
- Rate-of-change alarm detection (multi-reading trend analysis) — v2 enhancement via EventBridge
- Multi-signal correlation alarms (e.g., temperature + humidity combined threshold) — v2 via EventBridge pattern matching
- Step Functions for multi-step alarm escalation workflows (warn → escalate → page on-call) — Phase 4 or v2
- Amazon Managed Grafana as real-time monitoring dashboard connected to Timestream — Phase 4 consideration
- DynamoDB DAX caching layer for high-frequency device metadata lookups — v2 optimization
</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROC-01 | Architecture documents Lambda-based message processing with type-dependent routing | Lambda batch consumer pattern from Kinesis confirmed; type-routing via Rules Engine upstream already documented in Phase 1. Lambda processes telemetry batches and enriches with DynamoDB metadata. |
| PROC-02 | Architecture documents SQS dead-letter queues for failed processing (no silent data loss) | DLQ pattern confirmed: SQS DLQ with maxReceiveCount=3, 14-day retention, CloudWatch alarm on ApproximateNumberOfMessagesVisible > 0. Applies to both Lambda consumers. |
| PROC-03 | Architecture documents batched processing via Kinesis (not per-message Lambda invocation) for telemetry hot path | Kinesis ESM (Event Source Mapping) batch size 100-500 records, bisect-on-error strategy, parallelization factor, and cost comparison table all confirmed. |
| STOR-01 | Architecture documents Amazon Timestream for time-series telemetry (hot storage with memory + magnetic tiers) | Timestream LiveAnalytics confirmed: memory store (configurable hours/days), magnetic store (configurable months/years). Writes: $0.50/million. VPC Interface endpoint required. |
| STOR-02 | Architecture documents DynamoDB for device metadata, state, and latest-value cache | DynamoDB On-Demand confirmed for IoT variable workloads. TTL for dedup records and command queue entries. Gateway endpoint (free) for VPC access. Five table roles documented. |
| STOR-03 | Architecture documents Aurora Serverless v2 for user/role/alert-rule relational data | Aurora Serverless v2 confirmed: scales to 0.5 ACU minimum when idle, PostgreSQL dialect, deployed in private data subnets. RDS Proxy recommended for Lambda connection pooling. |
| STOR-04 | Architecture includes comparison table of storage alternatives with pros/cons and recommendation | Four-way comparison documented: Timestream vs DynamoDB time-series vs InfluxDB on EC2 vs RDS time-series; Aurora Serverless v2 vs RDS Provisioned vs DynamoDB vs Aurora Provisioned. |
| ALRM-01 | Architecture documents alarm detection via IoT Rules Engine threshold evaluation | Rules Engine SQL with WHERE clause confirmed. AlarmRule uses `SELECT *, topic(2) AS thingName FROM 'devices/+/alarm' WHERE value > threshold` (already established in Phase 1 routing table). |
| ALRM-02 | Architecture documents SNS fan-out for email notifications | SNS Standard topic → Lambda subscription → SES email confirmed. SES for HTML templates, bounce handling. SNS chosen over direct SES for extensibility. |
| ALRM-03 | Architecture documents EventBridge for extensible multi-channel notification routing | EventBridge rule pattern from SNS → EventBridge Pipes or direct Lambda→EventBridge PutEvents confirmed. Future targets: SMS, webhook, Slack, PagerDuty via EventBridge Pipes or HTTP API destination. |
| ALRM-04 | Architecture documents alarm deduplication strategy (prevent notification storms) | DynamoDB dedup table pattern confirmed: composite key (deviceId + alarmType), TTL = 15 minutes, conditional write on first occurrence within window, Lambda evaluator checks before SNS publish. |
</phase_requirements>

---

## Summary

Phase 2 documents three architectural domains that build directly on the IoT Core ingestion layer established in Phase 1: the telemetry hot-path processing pipeline (Kinesis → Lambda → storage writes), the multi-tier storage layer (Timestream, DynamoDB, Aurora Serverless v2), and the alarm detection and notification pipeline (IoT Rules Engine → Lambda evaluator → SNS → SES with EventBridge extensibility).

All locked decisions are well-supported by prior research. The CLAUDE.md technology stack table contains exact service selections, alternatives considered, and cost data — these are the authoritative reference for every table the planner produces. The Phase 1 output documents (`01-security-foundation.md`, `02-device-connectivity-ingestion.md`) establish the VPC topology, IAM roles, Kinesis configuration, and IoT Rules Engine SQL that Phase 2 documentation extends. No locked decision conflicts with prior architecture decisions.

The two most evaluator-critical items are: (1) the Kinesis batch-processing comparison table with concrete cost/throughput numbers (Success Criteria #1), and (2) the alarm deduplication strategy with DLQ coverage for both Lambda consumers (Success Criteria #4, #5). Both are fully resolvable from existing research without additional external queries.

**Primary recommendation:** Write Phase 2 documentation as three discrete markdown sections — processing pipeline, storage layer, alarm pipeline — each following the Phase 1 format: narrative → component inventory → Mermaid diagram → comparison table → design notes. Reference D-XX decision IDs inline. Start from the Kinesis endpoint already documented in `02-device-connectivity-ingestion.md` to maintain document continuity.

---

## Standard Stack

### Core Services for Phase 2

| Service | Version/Tier | Purpose | Why Standard |
|---------|--------------|---------|--------------|
| Amazon Kinesis Data Streams | On-Demand | Telemetry hot-path buffer | Already configured in Phase 1. Batch Lambda consumer (ESM) amortizes cold starts across 100-500 records. 24h replay, per-device ordering via thingName partition key. |
| AWS Lambda | Python 3.12 / arm64 (Graviton2) | Batch telemetry processor + alarm evaluator | Two separate functions with distinct IAM roles and DLQ configurations. Stateless; scales automatically from Kinesis trigger. arm64 is 20% cheaper with same concurrency. |
| Amazon SQS | Standard queue | Dead-letter queue for both Lambda consumers | SQS Standard (not FIFO) for DLQ — ordering not required for error handling. maxReceiveCount=3 before routing to DLQ. 14-day retention provides debugging window. |
| Amazon Timestream for LiveAnalytics | Serverless | Hot time-series telemetry store | Memory tier: configurable (hours to days), magnetic tier: configurable (months to years). Purpose-built for IoT range queries. Writes batched via Lambda SDK `WriteRecords` API. |
| Amazon DynamoDB | On-Demand | Five roles: device metadata, latest-value cache, command queue, alarm dedup, config state | TTL enables free deletion of transient records (dedup entries, command queue entries). Gateway VPC endpoint (free). Single-digit millisecond lookups. |
| Amazon Aurora PostgreSQL Serverless v2 | Serverless v2 (0.5–8 ACU) | Relational: users, roles, alert rule configs, device groups | Scales to 0.5 ACU minimum (not zero — clarification below). Private data subnet only. RDS Proxy recommended for Lambda connection reuse. |
| Amazon SNS | Standard topic | Alarm notification fan-out hub | One topic fans out to: Lambda (→ SES email), future SMS, webhook, EventBridge. Decouples alarm evaluator from notification delivery details. $0.50/million publishes. |
| Amazon SES | Managed email | Transactional alarm emails and reports | HTML template support, bounce/complaint tracking. Requires domain verification. $0.10/1K emails. Called from Lambda subscribed to SNS. |
| Amazon EventBridge | Serverless events | Future notification channel extensibility layer | Lambda alarm evaluator publishes event to EventBridge custom bus after SNS publish. EventBridge rules fan-out to future targets without modifying alarm evaluator code. |

### Supporting Services

| Service | Version/Tier | Purpose | When to Use |
|---------|--------------|---------|-------------|
| AWS RDS Proxy | Managed | Aurora connection pooling for Lambda | Required when Lambda concurrency > 50 — prevents Aurora connection exhaustion. Lambda cold starts create new DB connections; RDS Proxy reuses them. |
| Amazon CloudWatch | Managed | DLQ depth alarms, Lambda error rates, Timestream write latency | Alarm on DLQ `ApproximateNumberOfMessagesVisible > 0` for both telemetry and alarm DLQs. |
| AWS Secrets Manager | Managed | Aurora credentials retrieved by Lambda at startup | Lambda in VPC retrieves secret by ARN. Auto-rotation for Aurora credentials. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Timestream for time-series | DynamoDB with time-sorted SK | DynamoDB requires custom schema design (PK=deviceId, SK=timestamp); range queries and aggregations are less efficient. Use DynamoDB if already heavily invested in it and need sub-millisecond point lookups. |
| Aurora Serverless v2 | DynamoDB for relational data | DynamoDB can store user/role data but requires aggressive denormalization; no native joins; alert rule configs with complex conditions are harder to query. Use DynamoDB if fully NoSQL architecture is a hard requirement. |
| SNS fan-out | Direct Lambda → SES | Direct SES invocation from alarm evaluator works but couples the evaluator to email-only delivery. Any new channel (SMS, webhook) requires code changes to the evaluator. SNS decouples this. |
| EventBridge as extensibility layer | Step Functions | Step Functions is better for stateful multi-step escalation workflows (warn → escalate → page on-call). EventBridge is lighter for stateless rule-based routing. Deferred to v2 if escalation workflows are required. |

---

## Architecture Patterns

### Pattern 1: Kinesis Batch Lambda Consumer (ESM)

**What:** Lambda is triggered by Kinesis Data Streams via Event Source Mapping (ESM). Records are delivered in batches of 100-500. Lambda processes all records in the batch, writes to Timestream and DynamoDB, then checkpoints.

**When to use:** Always for high-volume telemetry where per-message invocation would be cost-prohibitive.

**Key ESM parameters:**
- `BatchSize`: 200 (within the locked 100-500 range; specific value is Claude's discretion)
- `BisectBatchOnFunctionError: true` — on Lambda failure, Kinesis splits the batch and retries each half independently, isolating the bad record
- `DestinationConfig.OnFailure.Destination`: SQS DLQ ARN — failed batches (after bisect exhaustion) route here
- `MaximumRetryAttempts`: 3 — after 3 attempts the record set routes to DLQ
- `ParallelizationFactor`: 1 (can increase to 10 for higher throughput per shard)

**Comparison table (required by D-03, Success Criteria #1):**

| Approach | Throughput | Cost at 1K devices/hourly | Replay | Ordering | Operational Complexity |
|----------|-----------|--------------------------|--------|----------|----------------------|
| IoT Rules → Lambda (per-message) | 1 invocation/message; throttles at concurrency limit | High — 1,000 invocations/hour minimum; cold starts on each spike | None — lost if Lambda fails | None | Low setup, high cost at scale |
| IoT Rules → SQS → Lambda | 1 invocation per SQS batch (up to 10 messages) | Medium — SQS + Lambda; FIFO adds cost | 4–14 days message retention; no stream replay | FIFO optional (costly) | Medium — SQS queue management |
| IoT Rules → Kinesis → Lambda (batch) | 1 invocation per 100-500 records; linear scaling | **Low** — 10-20x fewer invocations; Graviton2 reduces compute cost | **Yes** — 24h–7d stream replay from any position | Per-shard, by partition key | Medium — ESM config, shard management |

**Confidence:** HIGH (confirmed against AWS Lambda → Kinesis ESM documentation and Phase 1 research)

### Pattern 2: JSON Transformation with DynamoDB Enrichment

**What:** Lambda batch processor normalizes raw device JSON into a Timestream-compatible record format, enriching each record with device metadata from DynamoDB (device type, firmware version, location) to avoid re-lookup at query time.

**Transformation example (required by D-02):**

Raw device payload (from Kinesis record):
```json
{
  "temperature": 85.2,
  "humidity": 67,
  "ts": 1711500000,
  "thingName": "sensor-001"
}
```

DynamoDB device-metadata lookup (by thingName):
```json
{
  "thingName": "sensor-001",
  "deviceType": "temperature-sensor",
  "location": "warehouse-A",
  "firmwareVersion": "2.3.1"
}
```

Normalized Timestream write record:
```json
{
  "Dimensions": [
    {"Name": "thingName",       "Value": "sensor-001"},
    {"Name": "deviceType",      "Value": "temperature-sensor"},
    {"Name": "location",        "Value": "warehouse-A"}
  ],
  "MeasureName": "temperature",
  "MeasureValue": "85.2",
  "MeasureValueType": "DOUBLE",
  "Time": "1711500000000"
}
```

DynamoDB latest-value cache write (same Lambda invocation):
```json
{
  "PK": "DEVICE#sensor-001",
  "SK": "LATEST",
  "temperature": 85.2,
  "humidity": 67,
  "ts": 1711500000,
  "ttl": 1711503600
}
```

**Confidence:** HIGH (Timestream WriteRecords API schema confirmed; DynamoDB enrichment pattern confirmed from ARCHITECTURE.md)

### Pattern 3: DLQ Configuration for All Lambda Consumers

**What:** Both the telemetry processor Lambda and the alarm evaluator Lambda have SQS dead-letter queues configured. Failed records route to the DLQ after maxReceiveCount retries. CloudWatch alarms monitor DLQ depth.

**Required by D-04, Success Criteria #5.**

| Consumer Lambda | DLQ Name | maxReceiveCount | Retention | CloudWatch Alarm |
|-----------------|----------|-----------------|-----------|------------------|
| telemetry-processor | sqs-telemetry-dlq | 3 | 14 days | `ApproximateNumberOfMessagesVisible > 0` → SNS alert to ops |
| alarm-evaluator | sqs-alarm-dlq | 3 | 14 days | `ApproximateNumberOfMessagesVisible > 0` → SNS alert to ops |

For Kinesis ESM consumers, the DLQ is configured via `DestinationConfig.OnFailure` on the ESM, not on the Lambda function configuration itself. This distinction matters: Lambda function DLQ configuration applies to async invocations; ESM DLQ applies to stream/queue consumers.

**Confidence:** HIGH (confirmed against AWS Lambda ESM documentation)

### Pattern 4: Alarm Deduplication via DynamoDB TTL

**What:** Lambda alarm evaluator performs a conditional write to a DynamoDB dedup table before publishing to SNS. The write uses a composite key of `deviceId + alarmType` with a TTL of 900 seconds (15 minutes). If the item already exists (duplicate alarm within the window), the conditional write fails silently and SNS is not invoked.

**DynamoDB dedup table schema:**
```
PK: DEDUP#{thingName}#{alarmType}     (e.g., DEDUP#sensor-001#temperature_high)
TTL: unix_timestamp + 900              (auto-deleted after 15 minutes)
firstAlarmTs: unix_timestamp
alarmValue: 85.2
```

**Conditional write:** `ConditionExpression = "attribute_not_exists(PK)"`

If the condition fails (item exists → duplicate within 15 min window): skip SNS publish, increment a CloudWatch metric `AlarmsDeduplicated`.

If the condition succeeds (new alarm or window expired): write item, publish to SNS.

**Confidence:** HIGH (DynamoDB conditional writes + TTL pattern is standard; confirmed against DynamoDB Developer Guide)

### Pattern 5: SNS Fan-out → SES Email with EventBridge Extensibility

**What:** SNS Standard topic receives alarm event. Lambda subscription processes it and calls SES `SendTemplatedEmail`. EventBridge custom bus receives a copy of the processed alarm event for future channel routing.

**Fan-out topology:**
- SNS `alarm-notifications` topic → Lambda `ses-email-sender` (current)
- Lambda `alarm-evaluator` also calls `eventbridge:PutEvents` on custom bus `iot-alarm-bus` after SNS publish (extensibility hook)
- EventBridge rules on `iot-alarm-bus` can target: Lambda (SMS via SNS SMS), HTTP API destination (webhook), EventBridge Pipes → Slack/PagerDuty

**Why SNS over direct Lambda→SES:** SNS decouples the alarm evaluator from notification delivery. Adding a new channel requires only a new SNS subscription or EventBridge rule — no changes to alarm evaluator code.

**Confidence:** HIGH (SNS fan-out pattern confirmed; EventBridge Pipes as channel extensibility confirmed against AWS EventBridge docs)

### Recommended Documentation Structure for Phase 2 Output

```
docs/architecture/04-data-pipeline-processing.md
  ├── Kinesis → Lambda processor narrative
  ├── Batch processing comparison table (D-03)
  ├── JSON transformation example (D-02)
  ├── Hot-path Mermaid diagram (D-14)
  └── DLQ configuration table (D-04)

docs/architecture/05-storage-layer.md
  ├── Three-tier storage narrative
  ├── Timestream (memory/magnetic tiers, VPC endpoint)
  ├── DynamoDB (5 table roles, TTL, VPC endpoint)
  ├── Aurora Serverless v2 (VPC, RDS Proxy)
  ├── Storage comparison tables (D-07)
  └── VPC placement diagram (referencing Phase 1 topology)

docs/architecture/06-alarm-notifications.md
  ├── Alarm pipeline narrative
  ├── IoT Rules Engine SQL threshold detection (D-08)
  ├── Lambda alarm evaluator + dedup strategy (D-09)
  ├── SNS fan-out + SES email (D-10)
  ├── EventBridge extensibility (D-11)
  ├── Alarm pipeline Mermaid diagram (D-15)
  └── DLQ configuration for alarm evaluator (D-04)
```

### Anti-Patterns to Avoid

- **Configuring DLQ on Lambda function instead of ESM for Kinesis consumers:** Lambda function-level DLQ handles async invocations; Kinesis ESM `DestinationConfig.OnFailure` handles stream consumer failures. Both are needed — do not confuse them.
- **Alarm flooding via IoT Rule without WHERE clause:** Rules Engine routing with no threshold filter sends every reading to the alarm Lambda. The WHERE clause in the Rule SQL (`WHERE value > threshold`) is the first deduplication gate.
- **Timestream writes without batching:** Timestream charges per write request; sending one record at a time from Lambda multiplies write unit cost. Use `WriteRecords` with all records from the Kinesis batch in a single API call.
- **Aurora Serverless v2 "scales to zero" misconception:** Aurora Serverless v2 scales to a minimum of 0.5 ACU, not zero. It does NOT completely stop billing when idle (unlike DynamoDB On-Demand). This should be stated clearly in the cost section to be accurate with evaluators.
- **Missing RDS Proxy for Lambda → Aurora:** Lambda cold starts create new TCP connections to Aurora. Without RDS Proxy, concurrent Lambda invocations can exhaust Aurora's connection limit (~300 for small ACU configurations). RDS Proxy is required in the architecture diagram even if it adds cost.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Telemetry batch delivery to S3 | Custom Lambda batching + S3 PutObject logic | Kinesis Data Firehose (already in Phase 1) | Firehose handles buffering, retry, Parquet conversion, dynamic partitioning — 200+ lines of custom code replaced |
| Alarm deduplication with expiry | Custom Redis/cache-based dedup | DynamoDB TTL conditional write | DynamoDB TTL auto-deletes at no cost; conditional write is atomic; no cache cluster to manage |
| Database credential rotation | Custom cron + secret update | AWS Secrets Manager + RDS Proxy integration | Auto-rotation for Aurora; Lambda retrieves via SDK; no plaintext credentials anywhere |
| Email template rendering | Custom HTML builder in Lambda | SES templated emails (`SendTemplatedEmail`) | SES stores templates, handles bounces, complaint tracking, and delivery status — free to use within SES pricing |
| Multi-channel notification routing | If/else chains in alarm evaluator | SNS subscriptions + EventBridge rules | Each channel is an independent subscription/rule; zero code changes to evaluator when adding channels |
| Kinesis consumer checkpointing | Manual DynamoDB checkpoint tracking | Lambda ESM with `BisectBatchOnFunctionError` | ESM manages checkpointing automatically; bisect-on-error isolates poison-pill records without custom logic |

**Key insight:** This domain (IoT telemetry processing and notifications) has mature AWS-native solutions for every non-trivial problem. Any custom implementation of batching, deduplication, retry, or routing logic will be inferior to the managed service equivalent and will signal to evaluators that the design chose complexity over correctness.

---

## Common Pitfalls

### Pitfall 1: Silent Alarm Flooding (Missing WHERE Clause in IoT Rule)

**What goes wrong:** IoT AlarmRule routes ALL messages from `devices/+/alarm` to Lambda without a threshold condition. Every device publish on the alarm topic — including status updates, telemetry that happens to use the alarm topic, or test messages — triggers the full alarm pipeline.

**Why it happens:** The WHERE clause in IoT Rules Engine SQL is easy to omit when copying rule templates. The rule "works" in testing with a single device but floods production with thousands of notifications per hour.

**How to avoid:** AlarmRule SQL must include a WHERE clause: `SELECT *, topic(2) AS thingName, timestamp() AS ruleTimestamp FROM 'devices/+/alarm' WHERE value > threshold`. Threshold can be a literal value in the rule SQL, or better, the Lambda evaluator fetches the per-device threshold from Aurora (alert rule table) and applies it there.

**Warning signs:** Alarm notification pipeline described without mention of WHERE clause or Lambda threshold evaluation logic.

### Pitfall 2: Lambda DLQ Misconfiguration for Kinesis Consumers

**What goes wrong:** Developer sets `DeadLetterConfig` on the Lambda function itself. For Kinesis event source consumers, this has NO effect — Kinesis ESM failures must be handled via `DestinationConfig.OnFailure` on the Event Source Mapping resource, not the Lambda function configuration.

**Why it happens:** AWS documentation covers both Lambda async DLQ (function-level) and ESM DLQ (mapping-level) in different sections. Architects specify "Lambda with DLQ" without distinguishing the two configuration paths.

**How to avoid:** For Kinesis consumers, configure `DestinationConfig.OnFailure.Destination: arn:aws:sqs:...:sqs-telemetry-dlq` on the ESM resource. For alarm evaluator (invoked async by IoT Rules Engine), configure `DeadLetterConfig` on the Lambda function configuration. Both patterns must appear in the documentation.

**Warning signs:** DLQ described as "Lambda DLQ" without specifying ESM vs function-level configuration.

### Pitfall 3: Aurora Serverless v2 Minimum Cost Misrepresentation

**What goes wrong:** Architecture documentation states Aurora Serverless v2 "scales to zero" or "has no idle cost." Aurora Serverless v2 scales to a minimum of 0.5 ACU, billing approximately $0.06/hour (~$43/month minimum). This is clearly different from DynamoDB On-Demand's true zero-cost-at-zero-traffic model.

**Why it happens:** AWS marketing materials emphasize "scales down automatically" without always clarifying the 0.5 ACU minimum.

**How to avoid:** State explicitly: "Aurora Serverless v2 scales to a minimum of 0.5 ACU when idle — not zero. Monthly minimum cost: ~$0.06/hour × 720 hours = ~$43/month." This is honest and evaluators will respect the accuracy.

**Warning signs:** Cost analysis section shows $0 for Aurora at idle load.

### Pitfall 4: Timestream Memory Store Retention Confusion

**What goes wrong:** Architecture specifies Timestream memory store retention as "1 day" without understanding that records are NOT deleted at memory expiry — they are automatically tiered to the magnetic store. Specifying a very short memory retention (e.g., 1 hour) means dashboard queries for data older than 1 hour hit the magnetic store, which is slower and costs $0.01/GB vs $0/GB for memory queries.

**Why it happens:** Timestream documentation describes two stores separately; the automatic tiering behavior (not deletion) is easy to miss.

**How to avoid:** Recommend memory store retention of 24 hours (covers most dashboard lookups) with magnetic store retention of 365 days (1 year of historical data). State explicitly: "Data is tiered from memory to magnetic automatically — not deleted. Memory tier queries have no per-query cost; magnetic tier queries cost $0.01/GB scanned."

**Warning signs:** Architecture describes Timestream retention without explaining memory→magnetic tiering.

### Pitfall 5: DynamoDB Dedup Table Hot Partition

**What goes wrong:** Alarm dedup records use `deviceId` as the sole partition key. During alarm storms (many devices triggering simultaneously), all writes converge on a small number of partitions, causing throttling.

**Why it happens:** Simple composite key design looks reasonable for single-device testing.

**How to avoid:** Use a composite key that distributes writes: `PK = DEDUP#{thingName}#{alarmType}` — this ensures each device+alarm-type combination is a separate partition key, spreading writes across the DynamoDB partition space. With On-Demand billing, DynamoDB auto-scales throughput, but the partition key design determines actual throughput distribution.

**Warning signs:** Dedup table design shows only `deviceId` as PK without `alarmType` or timestamp component.

---

## Code Examples

Verified patterns from official sources and established Phase 1 patterns:

### IoT Rules Engine AlarmRule SQL (with threshold)
```sql
-- Source: AWS IoT Rules Engine SQL Reference
SELECT
  *,
  topic(2) AS thingName,
  timestamp() AS ruleTimestamp
FROM 'devices/+/alarm'
WHERE value > 80
```
Note: In production, the Lambda evaluator applies per-device thresholds from Aurora, making the WHERE clause a coarse pre-filter to reduce Lambda invocations.

### Lambda ESM Configuration (key parameters)
```json
{
  "EventSourceArn": "arn:aws:kinesis:us-east-1:123456789:stream/telemetry-stream",
  "FunctionName": "telemetry-processor",
  "BatchSize": 200,
  "BisectBatchOnFunctionError": true,
  "MaximumRetryAttempts": 3,
  "DestinationConfig": {
    "OnFailure": {
      "Destination": "arn:aws:sqs:us-east-1:123456789:sqs-telemetry-dlq"
    }
  }
}
```

### DynamoDB Dedup Conditional Write (pseudo-code)
```python
# Source: DynamoDB Developer Guide — Conditional Expressions
try:
    table.put_item(
        Item={
            "PK": f"DEDUP#{thing_name}#{alarm_type}",
            "TTL": int(time.time()) + 900,  # 15 minutes
            "firstAlarmTs": event_ts,
            "alarmValue": value
        },
        ConditionExpression="attribute_not_exists(PK)"
    )
    # Condition succeeded → first alarm in window → publish to SNS
    sns.publish(TopicArn=ALARM_TOPIC_ARN, Message=json.dumps(alarm_payload))
except ConditionalCheckFailedException:
    # Duplicate alarm within 15-minute window → suppress
    cloudwatch.put_metric_data(Namespace="IoT/Alarms", MetricData=[
        {"MetricName": "AlarmsDeduplicated", "Value": 1, "Unit": "Count"}
    ])
```

### Timestream WriteRecords Batch Pattern
```python
# Source: Amazon Timestream Developer Guide
records = []
for record in kinesis_batch:
    records.append({
        "Dimensions": [
            {"Name": "thingName", "Value": record["thingName"]},
            {"Name": "deviceType", "Value": metadata[record["thingName"]]["deviceType"]}
        ],
        "MeasureName": "temperature",
        "MeasureValue": str(record["temperature"]),
        "MeasureValueType": "DOUBLE",
        "Time": str(record["ts"] * 1000)  # milliseconds
    })
# One API call for entire batch — minimizes write unit cost
timestream_write.write_records(
    DatabaseName="iot-telemetry",
    TableName="device-metrics",
    Records=records
)
```

### EventBridge PutEvents from Alarm Evaluator
```python
# Source: Amazon EventBridge Developer Guide
eventbridge.put_events(
    Entries=[{
        "Source": "iot.alarm-evaluator",
        "DetailType": "AlarmTriggered",
        "Detail": json.dumps({
            "thingName": thing_name,
            "alarmType": alarm_type,
            "value": value,
            "threshold": threshold,
            "ts": event_ts
        }),
        "EventBusName": "iot-alarm-bus"
    }]
)
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Kinesis Data Analytics (SQL) for stream processing | AWS Managed Service for Apache Flink (or Lambda for simple transforms) | 2023 (KDA deprecated) | Do NOT reference Kinesis Data Analytics in the document — it is deprecated |
| Amazon IoT Analytics for IoT ETL | AWS Glue + S3 + Athena | April 2023 (IoT Analytics deprecated) | Do NOT reference IoT Analytics in the document |
| SNS email subscription (plain text) for notifications | SNS → Lambda → SES with HTML templates | Ongoing best practice | SNS email subscriptions lack HTML templates and programmatic unsubscribe |
| Kinesis Firehose minimum 60-second buffer | Zero Buffering feature (December 2023) enables ~5-second delivery | December 2023 | For near-real-time S3 delivery requirements; standard buffering still recommended for batch efficiency |
| Lambda DLQ on function config | ESM `DestinationConfig.OnFailure` for stream consumers | 2019 (ESM destinations added) | Critical distinction — function-level DLQ does not apply to Kinesis/SQS ESM consumers |

**Deprecated/outdated:**
- Amazon IoT Analytics: Service deprecated April 2023. AWS stopped accepting new customers. Replaced by Glue + S3 + Athena.
- Kinesis Data Analytics (legacy SQL): Deprecated. Use Managed Service for Apache Flink or Lambda.
- Amazon Timestream (original): Rebranded to Timestream for LiveAnalytics in 2023. Current product name in documentation.

---

## Open Questions

1. **Aurora Serverless v2 minimum ACU at idle**
   - What we know: AWS documentation states 0.5 ACU minimum (approximately $0.06/hour in us-east-1 as of 2026-03-27)
   - What's unclear: Whether zero-scale (0 ACU) is available in any configuration — it is NOT available in standard Serverless v2; Aurora Serverless v1 had zero-scale but is not recommended for new workloads
   - Recommendation: Document as "scales to 0.5 ACU minimum" — do not claim zero idle cost. LOW risk item but evaluators will notice if cost table claims $0 idle.

2. **Timestream VPC endpoint availability in all regions**
   - What we know: Timestream for LiveAnalytics VPC Interface endpoint is available in major AWS regions (us-east-1, us-west-2, eu-west-1)
   - What's unclear: Full regional availability — this is architecture documentation for a design exercise, so us-east-1 (the reference region from Phase 1) is confirmed supported
   - Recommendation: Proceed with us-east-1 as the reference region — Timestream VPC endpoint is confirmed available there.

3. **SNS → SES vs SNS email subscription for notification delivery**
   - What we know: SNS email subscriptions deliver plain-text emails without templates; SES allows HTML-formatted templates with unsubscribe management
   - What's unclear: Whether the design document should show the Lambda subscription to SNS calling SES, or show SES as a direct SNS subscription target
   - Recommendation: Document as Lambda subscription → SES (not direct SES subscription on SNS), because Lambda provides the template rendering and email customization step. This is also more extensible.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 2 is a documentation-only deliverable (architecture design document). No external tools, databases, CLIs, or runtimes are executed. The output is markdown files written to `docs/architecture/`. No environment dependency audit is required.

---

## Project Constraints (from CLAUDE.md)

The following directives from `CLAUDE.md` apply to Phase 2 planning and output:

| Directive | Applies To | Implication for Phase 2 |
|-----------|-----------|-------------------------|
| Platform: AWS primary — all services must be AWS-native | All service selections | No non-AWS services in primary recommendation (InfluxDB appears only in comparison table as rejected alternative) |
| Format: Documentation with Mermaid diagrams — must be readable without tooling | All output files | Two required Mermaid diagrams (D-14, D-15); no PlantUML or draw.io references |
| Device connectivity: Devices connect hourly for push only — architecture must queue commands | Alarm + processing design | Alarm evaluator must account for hourly-connect pattern in dedup window sizing (15 min > hourly rate, so one alarm per connect cycle is deduplicated correctly) |
| Data format: JSON ingestion, AI-ready format (Parquet) for Data Lake | Storage documentation | Phase 2 storage section confirms DynamoDB/Timestream for hot path; Parquet/S3 for cold path (already established, mentioned in storage tier documentation for completeness) |
| Security: Databases must be private (VPC-internal only) | Timestream, DynamoDB, Aurora | All three Phase 2 storage services placed in data subnets per Phase 1 VPC topology; VPC endpoint types explicitly stated |
| Scale target: Thousands of devices | Kinesis batch sizing, cost estimates | Cost table uses 1K–10K device range; Kinesis On-Demand mode handles scale without shard pre-provisioning |
| GSD Workflow Enforcement | All file edits | All output produced through GSD execute-phase workflow |

---

## Sources

### Primary (HIGH confidence)
- Phase 1 research: `.planning/research/STACK.md` — full service stack with alternatives; cost data for Timestream, DynamoDB, Aurora, Lambda, SNS, SES
- Phase 1 research: `.planning/research/ARCHITECTURE.md` — data flow patterns, component responsibilities, anti-patterns
- Phase 1 research: `.planning/research/PITFALLS.md` — Lambda DLQ configuration, alarm flooding, database VPC isolation
- Phase 1 output: `docs/architecture/01-security-foundation.md` — VPC topology, subnet inventory, IAM roles, VPC endpoints
- Phase 1 output: `docs/architecture/02-device-connectivity-ingestion.md` — IoT Rules Engine SQL, Kinesis configuration, existing processing diagram showing Lambda consumers
- CLAUDE.md project stack table — authoritative service selections with alternatives and cost optimization data
- [AWS Lambda Event Source Mapping — Kinesis](https://docs.aws.amazon.com/lambda/latest/dg/with-kinesis.html) — ESM configuration, BisectBatchOnFunctionError, DestinationConfig
- [Amazon Timestream for LiveAnalytics Developer Guide](https://docs.aws.amazon.com/timestream/latest/developerguide/) — WriteRecords API, memory/magnetic store tiering, VPC endpoint
- [DynamoDB Conditional Expressions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Expressions.ConditionExpressions.html) — Conditional write for dedup pattern
- [Amazon SNS Developer Guide](https://docs.aws.amazon.com/sns/latest/dg/) — Fan-out pattern, Lambda subscriptions
- [Amazon EventBridge Developer Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/) — Custom event buses, PutEvents, EventBridge Pipes

### Secondary (MEDIUM confidence)
- [Timestream IoT lessons learned — Superluminar (March 2025)](https://superluminar.io/2025/03/21/an-overview-and-lessons-learned-from-storing-and-analysing-iot-data-in-aws-timestream/) — Real-world Timestream pitfalls and write batching recommendations
- [How to ensure resilience for AWS IoT Rules Engine to Lambda integration — IoT Builders](https://dev.to/iotbuilders/how-to-ensure-resilience-for-your-aws-iot-rules-engine-to-aws-lambda-integration-1aoi) — errorAction + DLQ patterns

### Tertiary (LOW confidence — flagged for validation if needed)
- Aurora Serverless v2 minimum ACU pricing: Documented as $0.12/ACU-hour × 0.5 ACU minimum = $0.06/hour in CLAUDE.md stack table. Cross-verify against current AWS RDS pricing page before finalizing cost estimates.

---

## Metadata

**Confidence breakdown:**
- Processing pipeline (Kinesis batch, DLQ, ESM config): HIGH — confirmed from Phase 1 research + official AWS Lambda ESM docs
- Storage tier documentation (Timestream, DynamoDB, Aurora): HIGH — confirmed from CLAUDE.md stack table + official AWS service docs
- Alarm pipeline (Rules Engine SQL, SNS, SES, EventBridge): HIGH — confirmed from Phase 1 IoT Rules configuration + official AWS docs
- DLQ configuration distinction (ESM vs function-level): HIGH — confirmed from AWS Lambda ESM documentation
- Aurora Serverless v2 minimum ACU pricing: MEDIUM — consistent across sources but recommend spot-checking current RDS pricing page

**Research date:** 2026-03-27
**Valid until:** 2026-04-27 (30 days — stable AWS services; re-verify Aurora pricing if cost accuracy is critical)
