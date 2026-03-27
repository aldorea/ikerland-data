# Architecture Research

**Domain:** AWS IoT device monitoring platform (industrial/telemetry)
**Researched:** 2026-03-27
**Confidence:** HIGH (AWS official documentation + AWS whitepapers)

---

## Standard Architecture

### System Overview

```
┌──────────────────────────────────────────────────────────────────────────┐
│  DEVICE LAYER                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                      │
│  │  Device A   │  │  Device B   │  │  Device N   │  (MQTT / HTTPS)      │
│  │ (hourly)    │  │ (hourly)    │  │ (hourly)    │                      │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                      │
└─────────┼────────────────┼────────────────┼────────────────────────────┘
          │ MQTT TLS 8883  │                │
┌─────────▼────────────────▼────────────────▼────────────────────────────┐
│  INGESTION LAYER                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │                     AWS IoT Core (MQTT Broker)                    │   │
│  │   X.509 mutual TLS auth · Device Registry · Device Shadows        │   │
│  └────────────────────────────┬─────────────────────────────────────┘   │
│                               │ Rules Engine (SQL filter)               │
│             ┌─────────────────┼────────────────────┐                    │
│             ▼                 ▼                    ▼                    │
│  [Telemetry topic]  [Alarm topic]        [Config/Shadow topic]          │
└─────────────┼─────────────────┼────────────────────┼────────────────────┘
              │                 │                    │
┌─────────────▼─────────────────▼──────────────────┐ │
│  PROCESSING LAYER                                 │ │
│  ┌───────────────────┐  ┌──────────────────────┐  │ │
│  │  Amazon Kinesis   │  │  AWS Lambda          │  │ │
│  │  Data Streams     │  │  (alarm evaluation)  │  │ │
│  │  (telemetry fan-in│  └──────────┬───────────┘  │ │
│  │   buffer)         │             │ SNS/SES       │ │
│  └─────────┬─────────┘             ▼               │ │
│            │             ┌──────────────────────┐  │ │
│            │             │  Amazon SNS           │  │ │
│            │             │  (email + channels)   │  │ │
│            │             └──────────────────────┘  │ │
└────────────┼──────────────────────────────────────┘ │
             │                                         │
┌────────────▼─────────────────────────────────────────▼────────────────┐
│  STORAGE LAYER (VPC-private — no public access)                        │
│                                                                        │
│  HOT PATH (recent / dashboards)       COLD PATH (archive / ML)        │
│  ┌─────────────────────────────┐      ┌──────────────────────────────┐ │
│  │  Amazon Timestream          │      │  Amazon S3 (raw JSON)        │ │
│  │  · memory store: 1-7 days   │      │  · Firehose delivery stream  │ │
│  │  · magnetic store: 90 days  │      │  · partitioned by date+thing │ │
│  └─────────────────────────────┘      └──────────────┬───────────────┘ │
│  ┌─────────────────────────────┐                     │ Glue ETL        │
│  │  Amazon DynamoDB            │                     ▼                 │
│  │  · device metadata/state    │      ┌──────────────────────────────┐ │
│  │  · latest telemetry cache   │      │  Amazon S3 (Parquet)         │ │
│  │  · command queue            │      │  Data Lake — AI-ready        │ │
│  └─────────────────────────────┘      └──────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────┘
             │                  │
┌────────────▼──────────────────▼───────────────────────────────────────┐
│  API LAYER (inside VPC — no public DB access)                          │
│  ┌────────────────────────────────────────────────────────────────┐   │
│  │  Amazon API Gateway (REST)  ←→  AWS Lambda (business logic)    │   │
│  │  · Cognito authorizer       ←→  VPC Link to private resources  │   │
│  └────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
             │
┌────────────▼───────────────────────────────────────────────────────────┐
│  WEB FRONTEND                                                           │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  S3 Bucket (static assets)  ←  CloudFront CDN  ←  Route 53     │    │
│  │  · OAC (Origin Access Control) — S3 not public                 │    │
│  │  · WAF rules for protection                                    │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Component Responsibilities

| Component | Responsibility | AWS Service |
|-----------|----------------|-------------|
| MQTT Broker | Device connectivity, auth, topic routing | AWS IoT Core |
| Device Registry | Device identity, certificate management | AWS IoT Core Registry |
| Device Shadow | Desired/reported state store, command queue for offline devices | AWS IoT Device Shadow |
| Rules Engine | SQL-based message filtering and fan-out routing | AWS IoT Rules Engine |
| Stream buffer | High-throughput telemetry buffering (fan-in aggregation) | Amazon Kinesis Data Streams |
| Hot time-series store | Recent telemetry — dashboard queries, alerts | Amazon Timestream |
| Device state store | Metadata, latest values, command queue | Amazon DynamoDB |
| Cold archive | Raw JSON messages, long-term retention | Amazon S3 (via Kinesis Firehose) |
| ETL pipeline | JSON → Parquet conversion, partitioning for ML | AWS Glue |
| Data Lake | AI-ready Parquet, queryable with Athena | Amazon S3 + Glue Catalog |
| Alarm evaluator | Threshold evaluation, alert triggering | AWS Lambda |
| Notification fan-out | Email + channels delivery | Amazon SNS |
| API gateway | REST endpoint, auth, throttling | Amazon API Gateway |
| Business logic | API handlers, command dispatch | AWS Lambda (VPC) |
| Web hosting | Static SPA assets | Amazon S3 + CloudFront |
| DNS + TLS | Domain, certificate management | Route 53 + ACM |
| Auth | User pools, JWT tokens | Amazon Cognito |
| Secrets | DB credentials, API keys | AWS Secrets Manager |
| Observability | Logs, metrics, traces | CloudWatch + X-Ray |

---

## Data Flows

### Flow 1: Device Telemetry Ingestion (Hot Path)

```
Device
  │ MQTT PUBLISH dt/<app>/<ctx>/<thingName>/telemetry
  ▼
AWS IoT Core (TLS mutual auth)
  │ Rules Engine: SELECT * FROM 'dt/+/+/+/telemetry'
  ▼
Amazon Kinesis Data Streams
  │ Lambda consumer (batched)
  ▼
Amazon Timestream (hot — memory/magnetic store)
  │
  ├─── also → DynamoDB (latest-value cache per device, TTL-based)
  └─── also → S3 via Kinesis Firehose (raw JSON cold archive)
```

**Confidence:** HIGH — AWS IoT Core → Kinesis → Timestream is the recommended pattern per AWS Timestream docs.

### Flow 2: Alarm Pipeline

```
Device
  │ MQTT PUBLISH dt/<app>/<ctx>/<thingName>/alarm
  ▼
AWS IoT Core Rules Engine
  │ Rule: SELECT * FROM 'dt/+/+/+/alarm'
  ▼
AWS Lambda (alarm evaluator)
  │ Evaluates severity, deduplicates, enriches from DynamoDB
  ▼
Amazon SNS Topic
  ├─── Email subscription (SES/SNS)
  ├─── SMS subscription
  └─── Webhook subscription (HTTP endpoint)
```

**Confidence:** HIGH — SNS as notification fan-out is the canonical AWS pattern. EventBridge can replace SNS for complex routing rules.

**Alternative:** IoT Core can directly trigger SNS as a rule action for simple threshold alarms, bypassing Lambda when no enrichment is needed.

### Flow 3: Cold Archive and ETL to Data Lake

```
Kinesis Data Streams
  │
  ▼
Kinesis Data Firehose
  │ Buffer: 5 min / 128 MB
  │ Prefix: raw/year=!{timestamp:yyyy}/month=.../day=.../
  ▼
Amazon S3 (raw zone — JSON)
  │
  ▼ Glue Crawler (schema discovery)
  │
  ▼ AWS Glue ETL Job (scheduled — e.g., nightly)
  │ · JSON → Parquet conversion
  │ · Columnar partitioning: device_id, year, month, day
  │ · Medallion: Bronze (raw) → Silver (clean) → Gold (aggregated)
  ▼
Amazon S3 (processed zone — Parquet)
  │
  ▼ Glue Data Catalog + Amazon Athena (ad-hoc SQL queries)
```

**Confidence:** HIGH — AWS Glue + S3 Parquet is the recommended Data Lake ETL pattern per AWS Big Data blog.

### Flow 4: Command Delivery to Disconnected Devices

```
Web UI (operator sends command)
  │ POST /api/devices/{thingName}/commands
  ▼
API Gateway → Lambda
  │ Writes to Device Shadow: desired.{commandKey} = {value}
  │ Also writes to DynamoDB command queue (audit trail)
  ▼
AWS IoT Device Shadow (cloud state store)
  │ Persists desired state regardless of device connectivity
  │
  ──── Device is OFFLINE ───────────────────────────────────
  │
  Device reconnects (hourly)
  │ Subscribes to: $aws/things/{thingName}/shadow/update/delta
  ▼
Device receives delta document (desired vs reported diff)
  │ Executes command, updates local state
  ▼
Device PUBLISH: $aws/things/{thingName}/shadow/update
  │ Sets reported.{commandKey} = {value}
  ▼
Shadow reconciled → Lambda can detect completion via shadow update rule
```

**Confidence:** HIGH — Device Shadow desired/reported pattern is documented in AWS IoT Core Developer Guide.

**Key insight:** Version numbers in shadow documents prevent duplicate command execution when messages arrive out of order. Devices discard delta messages with older version numbers.

### Flow 5: Web Frontend Data Flow

```
Browser → CloudFront (global CDN, WAF, TLS)
  │ Static assets from S3 (OAC — bucket not publicly accessible)
  │
  ├─── API calls → API Gateway (REST) → Lambda → DynamoDB / Timestream
  │
  └─── Real-time updates (optional):
       API Gateway WebSocket API → Lambda → push telemetry snapshots
```

**Confidence:** HIGH — S3 + CloudFront with OAC is the current AWS recommended pattern (Origin Access Control replaced OAI in 2022).

### Flow 6: Authentication Flow

```
Browser → Cognito User Pool (login)
  │ Returns JWT (ID token + Access token)
  ▼
API Gateway (Cognito Authorizer validates JWT)
  │ Authorized requests reach Lambda
  ▼
Lambda → accesses DynamoDB / Timestream via IAM role (no credentials in code)
```

---

## Architectural Patterns

### Pattern 1: Fan-In via Rules Engine

**What:** Multiple devices publish to topic-per-device; Rules Engine uses a wildcard SQL subscription to aggregate all messages into a single Kinesis stream.

**When to use:** Always — it avoids MQTT connection limits on single subscribers and decouples ingestion from processing.

**Trade-offs:** Adds Kinesis cost; enables replay, backpressure handling, and Lambda consumers.

**Topic structure:**
```
dt/{app-prefix}/{context}/{thingName}/telemetry
dt/{app-prefix}/{context}/{thingName}/alarm
dt/{app-prefix}/{context}/{thingName}/config

Rules Engine SQL:
SELECT *, topic(4) AS thing_name FROM 'dt/iot-platform/+/+/telemetry'
```

### Pattern 2: Device Shadow for Async Command Delivery

**What:** Cloud writes desired state to Device Shadow; device reads the delta on reconnect and confirms via reported state.

**When to use:** Any time devices are intermittently connected. Avoids message loss — IoT Core persists the desired state until the device acknowledges it.

**Trade-offs:** Commands are not real-time (delivery delay = next connection cycle). Not suitable for sub-second control.

**Shadow document:**
```json
{
  "state": {
    "desired": { "sampleRate": 300, "reportEnabled": true },
    "reported": { "sampleRate": 60,  "reportEnabled": true },
    "delta":   { "sampleRate": 300 }
  },
  "version": 14
}
```

### Pattern 3: Hot/Cold Storage Split

**What:** Recent data (last 7–90 days) goes to Timestream (time-series optimized, fast queries). All data goes to S3 in raw form for archival and ML training.

**When to use:** Always — storing everything in Timestream is expensive; storing everything in S3 raw is slow for dashboards.

**Trade-offs:** Adds operational complexity; dual-write via Kinesis stream simplifies synchronization.

| Store | Data | Retention | Query Use |
|-------|------|-----------|-----------|
| Timestream (memory) | Latest 7 days | 1–7 days | Live dashboards, alarms |
| Timestream (magnetic) | Recent history | 90 days | Trend charts, device health |
| DynamoDB | Latest value per device | Indefinite (small) | Single-device lookups, API |
| S3 raw (JSON) | All messages | Configurable (lifecycle) | Audit, replay |
| S3 processed (Parquet) | Transformed | Indefinite | ML, Athena analytics |

### Pattern 4: Medallion Data Lake (Bronze/Silver/Gold)

**What:** S3 organized into three zones: raw (Bronze), cleaned and validated (Silver), aggregated and business-ready (Gold). Glue ETL jobs process each transition.

**When to use:** When the data lake must serve both ML engineers (raw data) and analysts (aggregated KPIs).

**Trade-offs:** More S3 storage cost; much better query performance and data quality.

### Pattern 5: CQRS via Lambda and DynamoDB

**What:** Write path (device ingestion → Kinesis → Timestream) is separate from read path (API Gateway → Lambda → DynamoDB/Timestream). DynamoDB caches the latest value per device so the API never hits Timestream for single-device lookups.

**When to use:** When read and write access patterns differ (bulk writes vs single-item reads).

**Trade-offs:** DynamoDB adds cost and eventual consistency delay for the latest value cache.

---

## Anti-Patterns

### Anti-Pattern 1: Fan-In to a Single Device

**What people do:** Have thousands of devices subscribe to each other's messages via shared MQTT topics on IoT Core.

**Why it's wrong:** Hits the non-adjustable per-connection message limit on IoT Core. Devices are not designed to be message aggregators.

**Do this instead:** Route all messages through the IoT Rules Engine into Kinesis or SQS. Let backend services aggregate.

### Anti-Pattern 2: Polling Device Shadow from Dashboard

**What people do:** Web dashboard polls the Device Shadow API every few seconds to show live device state.

**Why it's wrong:** Shadow is optimized for state sync on reconnect, not for real-time streaming. High poll frequency drives up IoT Core API costs.

**Do this instead:** Store the latest telemetry snapshot in DynamoDB (updated on each message arrival). Dashboard reads from DynamoDB via API Lambda. For real-time push, use API Gateway WebSocket.

### Anti-Pattern 3: Public Database Endpoints

**What people do:** Enable public access on DynamoDB, Timestream, or RDS for convenience.

**Why it's wrong:** Violates the explicit project security requirement and AWS Well-Architected Framework. Increases attack surface significantly.

**Do this instead:** All datastores in private VPC subnets. Lambda functions deployed into the same VPC with security groups. API Gateway uses VPC Link for private integrations.

### Anti-Pattern 4: Sending Commands via MQTT Directly (No Shadow)

**What people do:** Publish a command MQTT message directly to the device topic and expect the device to receive it.

**Why it's wrong:** If the device is offline (which it is, hourly-connection pattern), the message is lost immediately. MQTT retained messages on IoT Core have limitations and are not a reliable command queue.

**Do this instead:** Write desired state to Device Shadow. IoT Core persists it and delivers the delta when the device connects. For guaranteed ordered execution, add a DynamoDB command queue with status tracking.

### Anti-Pattern 5: Storing Raw JSON in S3 Without Partitioning

**What people do:** Dump all IoT messages as flat JSON files in a single S3 prefix.

**Why it's wrong:** Athena scans the entire bucket for every query — costs explode, performance degrades as data grows.

**Do this instead:** Use Kinesis Firehose with dynamic partitioning: `raw/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/thing=!{partitionKeyFromQuery:thingName}/`. Glue ETL converts to Parquet with the same partition scheme.

---

## Recommended Documentation Structure (Build Order)

The sections of the architecture document depend on each other in this order:

```
1. Network & Security Foundation
   └── VPC, subnets (public/private), security groups, IAM roles, ACM
       Required by: everything else

2. Device Connectivity Layer
   └── IoT Core, Certificate Authority, Device Registry, Thing types/groups
       Required by: Rules Engine, Device Shadow

3. Message Routing (Rules Engine)
   └── Topic structure, SQL rules, actions (Kinesis, Lambda, Firehose, Timestream)
       Required by: all downstream processing

4. Storage Layer
   └── Timestream (hot), DynamoDB (state + cache), S3 (cold archive)
       Required by: API layer, ETL pipeline

5. Device Shadow & Command Delivery
   └── Shadow service, desired/reported pattern, delta delivery
       Required by: API command endpoints

6. Alarm Pipeline
   └── Lambda alarm evaluator, SNS topics, subscriptions
       Required by: notification section

7. Data Lake ETL
   └── Kinesis Firehose, S3 raw zone, Glue Crawlers, Glue ETL, S3 Parquet zone
       Required by: Athena analytics, ML section

8. API Layer
   └── API Gateway, Lambda handlers, Cognito authorizer, VPC Link
       Required by: Web frontend

9. Web Frontend
   └── S3 bucket (SPA assets), CloudFront, OAC, Route 53, WAF
       Depends on: API layer for data

10. Observability
    └── CloudWatch dashboards, X-Ray tracing, IoT Core Fleet Indexing
        Depends on: all components above
```

---

## Scalability Considerations

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 0–1K devices | Single Kinesis shard (1 MB/s), DynamoDB on-demand, Lambda defaults — all serverless auto-scales |
| 1K–10K devices | Add Kinesis shards (1 shard per ~1K msg/s), DynamoDB provisioned + auto-scaling, Timestream magnetic store retention tuning |
| 10K–100K devices | Multi-shard Kinesis, Glue ETL job concurrency increase, Timestream scheduled queries for pre-aggregation, CloudFront + S3 static asset caching essential |
| 100K+ devices | IoT Core Fleet Provisioning for zero-touch device onboarding, Kinesis → MSK (Kafka) for more flexible stream processing, consider AWS IoT FleetWise for structured vehicle/industrial telemetry |

**First bottleneck:** Kinesis shard capacity (1 MB/s or 1K records/s per shard). Fix: shard splitting or switching to on-demand shards.

**Second bottleneck:** Lambda cold starts on API layer under bursty load. Fix: provisioned concurrency for API lambdas.

---

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Email delivery | SNS → SES or SNS email subscription | SNS email for low-volume; SES for branded HTML templates |
| SMS notifications | SNS SMS | Per-region limitations; costs per message |
| Webhook / third-party | SNS HTTP/S subscription | Target must respond with 200 within timeout |
| Azure / GCP equivalent | Azure IoT Hub / Google Cloud IoT Core + Pub/Sub | Bonus architecture note only |

### Internal Component Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| IoT Core ↔ Kinesis | IoT Rules Engine direct action | No Lambda in path — reduces latency and cost |
| Kinesis ↔ Timestream | Lambda consumer (event source mapping) | Batching reduces Timestream write units |
| Kinesis ↔ S3 | Kinesis Data Firehose (no code needed) | Dynamic partitioning via Firehose record transformation |
| Lambda ↔ DynamoDB | AWS SDK (IAM role, VPC endpoint) | VPC endpoint avoids public internet traversal |
| Lambda ↔ Timestream | AWS SDK (VPC endpoint for Timestream) | Use VPC endpoint to keep traffic private |
| API Gateway ↔ Lambda | Lambda proxy integration | Simplest; Lambda handles routing and response shaping |
| CloudFront ↔ S3 | Origin Access Control (OAC) | S3 bucket policy allows only CloudFront principal |
| Web UI ↔ API | HTTPS REST + JWT (Cognito) | Cognito Authorizer on API Gateway validates tokens |
| Lambda ↔ IoT Core (shadow write) | AWS SDK IoT Data Plane API | Lambda in VPC uses IoT Core VPC endpoint |

---

## Sources

- [AWS IoT Core Developer Guide — Device Shadow service](https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html) — HIGH confidence
- [Amazon Timestream — IoT Core integration](https://docs.aws.amazon.com/timestream/latest/developerguide/IOT-Core.html) — HIGH confidence
- [AWS Whitepaper — Designing MQTT Topics for AWS IoT Core](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/mqtt-communication-patterns.html) — HIGH confidence
- [AWS IoT Rules Engine documentation](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html) — HIGH confidence
- [AWS Blog — Patterns for IoT time-series ingestion with Timestream](https://aws.amazon.com/blogs/database/patterns-for-aws-iot-time-series-data-ingestion-with-amazon-timestream/) — HIGH confidence
- [AWS Blog — Build a Data Lake Foundation with AWS Glue and S3](https://aws.amazon.com/blogs/big-data/build-a-data-lake-foundation-with-aws-glue-and-amazon-s3/) — HIGH confidence
- [AWS Prescriptive Guidance — Deploy React SPA to S3 and CloudFront](https://docs.aws.amazon.com/prescriptive-guidance/latest/patterns/deploy-a-react-based-single-page-application-to-amazon-s3-and-cloudfront.html) — HIGH confidence
- [AWS Best Practices — API Gateway private APIs](https://docs.aws.amazon.com/whitepapers/latest/best-practices-api-gateway-private-apis-integration/rest-api.html) — HIGH confidence
- [IoT Atlas — Command via Device Shadow](https://iotatlas.net/en/implementations/aws/command/command2/) — MEDIUM confidence
- [Medallion Architecture on AWS — DEV Community](https://dev.to/aws-builders/medallion-architecture-on-aws-2ngm) — MEDIUM confidence

---

*Architecture research for: AWS IoT device monitoring platform*
*Researched: 2026-03-27*
