# Feature Research

**Domain:** AWS IoT device monitoring platform (industrial/telemetry, architecture design document)
**Researched:** 2026-03-27
**Confidence:** HIGH (core AWS services) / MEDIUM (pattern recommendations and tradeoffs)

---

## Feature Landscape

### Table Stakes (Evaluators Expect These)

Features an architecture evaluator will assume are present. Missing any of these signals incomplete AWS knowledge.

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| **MQTT-based device ingestion via AWS IoT Core** | Industry standard entry point for IoT on AWS; all well-designed AWS IoT platforms start here | LOW | MQTT over TLS 8883, HTTPS for publish-only, WebSocket for browser clients |
| **IoT Rules Engine for message routing** | Native AWS mechanism to filter/transform/route without additional middleware; well-known in AWS IoT | LOW | SQL-like conditions route to Lambda, SQS, Kinesis, S3, DynamoDB, SNS directly |
| **Device Shadow for command/config delivery to disconnected devices** | The canonical AWS solution for the "offline device" problem; evaluators will look for this explicitly | MEDIUM | Stores desired+reported+delta state; device reads delta on reconnect; solves the hourly-connect constraint |
| **Message type routing (telemetry vs config vs alarm events)** | Devices send mixed payloads; routing by type is baseline data hygiene | LOW | IoT Rules Engine topic-based or attribute-based routing |
| **Time-series storage for telemetry (Amazon Timestream)** | Purpose-built for sensor/telemetry data; evaluators know Timestream is the AWS-native answer here | MEDIUM | 1000x faster and 10x cheaper than relational for time-based queries; memory + magnetic tier auto-management |
| **Alarm/threshold notification system** | Core IoT monitoring requirement; without alerting there is no monitoring | MEDIUM | IoT Rules Engine detects condition → SNS topic → email/SMS/webhook fan-out |
| **REST API for the web application** | Required explicitly in the brief; must be AWS-native, scalable, and private-network aware | MEDIUM | API Gateway (HTTP API) + Lambda or ALB + ECS Fargate; all behind VPC |
| **Web dashboard for data visualization** | Explicitly required; must show real-time and historical telemetry | MEDIUM | React SPA on S3+CloudFront or Amazon Managed Grafana connected to Timestream |
| **S3-based Data Lake with Parquet ETL** | Explicitly required in the brief; AI/ML readiness via columnar format is now baseline expectation | HIGH | Raw zone (JSON) → Glue ETL job → Curated zone (Parquet, partitioned by device+date) |
| **VPC-isolated databases (no public access)** | Explicitly required; evaluators will check for private subnets and security groups | LOW | All RDS/DynamoDB/Timestream access routed through VPC endpoints or private subnets; no 0.0.0.0/0 |
| **X.509 certificate-based device authentication** | AWS IoT Core best practice; the expected authentication model for device fleets | LOW | Each device gets a unique cert; IoT policies restrict per-device MQTT topic access |
| **IAM least-privilege for all service-to-service communication** | AWS Well-Architected Security pillar baseline | LOW | Per-Lambda execution roles, no wildcard actions in production policies |
| **Encryption at rest and in transit** | Baseline compliance expectation; TLS 1.2+ enforced by IoT Core | LOW | KMS-managed keys for S3 (SSE-KMS), DynamoDB, Timestream; TLS enforced by AWS IoT Core |
| **CloudWatch logging and monitoring** | Operational baseline; expected for any production AWS architecture | LOW | IoT Core logs to CloudWatch; Lambda metrics; API Gateway access logs |

---

### Differentiators (Demonstrate Advanced Knowledge)

Features that go beyond checklist compliance and show architectural depth. An evaluator reviewing for a Cloud/AWS position will respond positively to these.

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| **Kinesis Data Firehose as ingestion buffer (vs direct Rules Engine → Lambda)** | Shows cost awareness and scalability thinking; 68% cheaper than direct Lambda invocation at scale; adds replay capability | MEDIUM | IoT Core → Rules Engine → Kinesis Data Firehose → S3 raw zone; Firehose handles batching, compression, S3 partitioning automatically |
| **SQS dead-letter queue (DLQ) on Lambda consumers** | Shows production-readiness; prevents silent data loss when processing fails | LOW | DLQ on each Lambda → alerts ops team; enables reprocessing without data loss |
| **Multi-tier S3 Data Lake (raw / processed / curated zones)** | Shows data engineering maturity; separates concerns between immutable audit trail and analytics-ready data | MEDIUM | Raw zone = original JSON (never deleted); Processed = normalized; Curated = Parquet partitioned by date/deviceId for Athena |
| **AWS Glue Data Catalog + Athena for ad-hoc queries** | Enables SQL-based exploration of the Data Lake without pre-provisioned infrastructure | MEDIUM | Glue Crawler discovers Parquet schema → Data Catalog → Athena runs serverless SQL; no ETL query infrastructure to manage |
| **S3 Intelligent-Tiering for Data Lake cost optimization** | Shows cost optimization depth; IoT archives grow unbounded; tiering avoids manual lifecycle management | LOW | Automatically moves objects between access tiers based on usage patterns; no retrieval fees for objects moving back to frequent access |
| **EventBridge for alarm event fan-out and extensibility** | Decouples alarm detection from notification delivery; allows adding new notification channels without touching alarm logic | MEDIUM | IoT Rules Engine → EventBridge event bus → multiple targets (SNS email, Lambda webhook, future Slack/PagerDuty integration) |
| **Amazon Managed Grafana for visualization (vs custom web app)** | Native AWS Timestream + Grafana integration; operational dashboards in hours not weeks; evaluators know this pattern | MEDIUM | AMG connects directly to Timestream data source; IoT SiteWise plugin available; avoids building custom charting from scratch |
| **AWS IoT Device Defender for anomaly detection** | Shows security depth beyond access control; ML-based behavioral baselining for compromised device detection | MEDIUM | Monitors MQTT behavior patterns; alerts on unusual publish rates, unauthorized topics, certificate misuse |
| **VPC endpoints (PrivateLink) for IoT Core credential provider** | Allows Greengrass or internal services to talk to IoT Core without traversing public internet | MEDIUM | Eliminates internet egress cost and attack surface for private device networks |
| **Glue ETL job triggered by S3 event (event-driven ETL)** | Shows preference for event-driven over polling-based; ETL runs only when new data arrives | LOW | S3 event notification → Lambda → Glue Job trigger; avoids scheduled jobs that run empty |
| **Device fleet grouping and thing types in IoT Core registry** | Shows operational maturity; enables policy-based management and bulk operations at scale | LOW | Thing groups allow bulk shadow updates; thing types enforce schema consistency |
| **Cost analysis section with per-service pricing model** | Explicitly valued in the brief; shows cloud economics thinking | HIGH | Per-message pricing (IoT Core), per-GB (Firehose), per-query (Athena), per-sample (Timestream) — all documented with scale projections |

---

### Anti-Features (Do Not Recommend These)

Patterns that seem appealing but create problems for this specific context. Recommending these in the architecture document would signal poor judgment to an evaluator.

| Anti-Feature | Why It Seems Appealing | Why It Is Problematic | Better Alternative |
|--------------|------------------------|-----------------------|--------------------|
| **Real-time streaming for all telemetry** | "Real-time" sounds better; Kinesis Data Streams is powerful | Devices connect hourly, not continuously — real-time infrastructure for batch-arriving data wastes cost; Kinesis Data Streams adds operational overhead (shard management, scaling) vs Firehose | Kinesis Data Firehose (serverless, automatic batching) or IoT Rules → SQS for near-real-time with lower cost |
| **Amazon RDS (relational) as primary telemetry store** | Familiar, ACID-compliant, SQL-queryable | Relational DBs are 10x more expensive and 1000x slower for time-series queries; schema migrations become painful as new sensor types are added | Amazon Timestream for telemetry; DynamoDB for device metadata and shadow state |
| **Polling the API for dashboard updates** | Simple to implement initially | At thousands of devices with hourly data, polling creates unnecessary load and latency spikes; breaks cost model | WebSocket via API Gateway + IoT Core MQTT-over-WebSocket for live updates in Grafana |
| **Single monolithic Lambda for all message types** | Simpler to write; one deployment unit | Mixed workloads (telemetry vs alarms vs config) have different SLAs, scaling needs, and error handling; a slow telemetry batch blocks alarm processing | Separate Lambda functions per message type, each with independent concurrency limits and DLQs |
| **Shared X.509 certificate across device fleet** | Easier provisioning; one cert to manage | If the shared cert is revoked (e.g., due to compromise), the entire fleet goes offline; violates least-privilege | Individual per-device certificates via IoT Core fleet provisioning or Just-In-Time Registration (JITR) |
| **Public-facing RDS or Timestream endpoints** | Simplifies connection strings for Lambda | Explicitly prohibited in the brief; violates AWS Well-Architected Security pillar; expands attack surface | All DBs in private subnets; Lambda in VPC with security groups; API Gateway as the only public-facing entry point |
| **AWS IoT Analytics (legacy service)** | Appears in older AWS IoT tutorials; has built-in pipeline features | AWS IoT Analytics is a legacy service that AWS no longer actively develops; its pipeline/dataset model is less flexible than Glue+Athena and costs more at scale | AWS Glue ETL + Athena + S3 (current recommended pattern) |
| **QuickSight as primary operational dashboard** | Fully managed, serverless BI with ML anomaly detection | QuickSight's SPICE engine is designed for business reporting (static snapshots), not live operational dashboards; time-series drill-down is less capable than Grafana; $18/user/month adds up | Amazon Managed Grafana for operational/engineering dashboards; QuickSight only if business stakeholder BI reporting is needed |
| **Over-provisioned EC2 for REST API** | Familiar deployment model; full control | Runs 24/7 even when devices connect only hourly; violates cost optimization principle for variable workloads | API Gateway + Lambda (serverless, pay-per-request) or ECS Fargate (containerized, scale-to-zero possible) |

---

## Feature Dependencies

```
[X.509 Device Authentication]
    └──required by──> [AWS IoT Core MQTT Ingestion]
                           └──required by──> [IoT Rules Engine Routing]
                                                  └──required by──> [All downstream pipelines]

[IoT Rules Engine Routing]
    ├──feeds──> [Kinesis Firehose → S3 Raw Zone]
    │               └──triggers──> [Glue ETL Job]
    │                                   └──produces──> [S3 Curated Zone (Parquet)]
    │                                                       └──queried by──> [Athena]
    ├──feeds──> [Amazon Timestream (telemetry)]
    │               └──visualized by──> [Amazon Managed Grafana]
    ├──feeds──> [Device Shadow updates]
    │               └──read by──> [Disconnected devices on reconnect]
    │               └──written by──> [REST API (command/config push)]
    └──feeds──> [Alarm detection rule]
                    └──triggers──> [EventBridge event]
                                       └──routes to──> [SNS → Email/SMS]

[VPC private networking]
    └──required by──> [All database access (Timestream, DynamoDB)]
    └──required by──> [Lambda → DB connections]

[REST API (API Gateway + Lambda)]
    └──depends on──> [Cognito or IAM for web app auth]
    └──writes to──> [Device Shadow (command delivery)]
    └──reads from──> [Timestream (telemetry history)]

[S3 Curated Zone (Parquet)]
    └──enhances──> [ML/AI readiness of Data Lake]
    └──enables──> [Glue Data Catalog + Athena ad-hoc queries]
```

### Dependency Notes

- **IoT Rules Engine requires IoT Core ingestion:** Rules only fire on messages received by IoT Core; the entire pipeline begins here.
- **Glue ETL requires S3 Raw Zone:** ETL reads from the raw JSON landing zone; raw zone must exist before curated zone can be created.
- **Device Shadow requires IoT Core:** Shadow service is part of IoT Core; there is no shadow without a registered Thing.
- **Alarm detection requires Timestream or Rules Engine:** Either evaluate thresholds in the Rules Engine (real-time) or query Timestream periodically (near-real-time); both patterns are valid.
- **Athena requires Glue Data Catalog:** Athena cannot query S3 Parquet without a schema registered in the Glue catalog; Glue Crawler must run first.
- **Private DB access requires Lambda in VPC:** Lambda functions accessing Timestream or DynamoDB in private subnets must themselves be placed in a VPC with appropriate security group rules.

---

## MVP Definition

This is a design document assessment, not a product launch. "MVP" here means: what is the minimum coherent architecture that demonstrates all required capabilities and can be documented completely.

### Document With (Core Architecture — Required by Brief)

- [ ] **IoT Core + MQTT ingestion** — entry point for all device data; defines the connectivity model
- [ ] **IoT Rules Engine for message type routing** — separates telemetry, config, and alarm event flows
- [ ] **Device Shadow for command/config delivery** — solves the disconnected device constraint explicitly
- [ ] **Amazon Timestream for telemetry storage** — purpose-built time-series; justifiable with comparison table
- [ ] **DynamoDB for device metadata and shadow state persistence** — fast key-value lookups for device registry
- [ ] **S3 Raw Zone + Glue ETL + Parquet Curated Zone** — Data Lake ETL to AI-ready format (explicitly required)
- [ ] **Glue Data Catalog + Athena** — enables Data Lake querying without additional infrastructure
- [ ] **Alarm detection via Rules Engine + SNS** — threshold evaluation + email notification (explicitly required)
- [ ] **API Gateway + Lambda for REST API** — serverless, scalable, aligns with variable IoT workload pattern
- [ ] **Amazon Managed Grafana or CloudFront SPA** — web visualization (explicitly required)
- [ ] **VPC private subnets for all databases** — security requirement (explicitly required)
- [ ] **X.509 per-device certificates + IAM least-privilege** — security baseline (explicitly required)
- [ ] **Architecture diagrams (Mermaid)** — the actual deliverable artifact
- [ ] **Technology comparison tables** — one per key decision (e.g., Timestream vs DynamoDB vs RDS; Grafana vs QuickSight; Kinesis vs SQS vs direct Lambda)

### Strengthen With (Differentiators — Demonstrate Depth)

- [ ] **Kinesis Firehose as ingestion buffer** — document cost comparison vs direct Lambda; add when discussing scaling
- [ ] **EventBridge for alarm fan-out** — add when designing notification extensibility
- [ ] **IoT Device Defender** — add in security section to show defense-in-depth
- [ ] **S3 Intelligent-Tiering lifecycle policy** — add in cost optimization section
- [ ] **SQS DLQ on Lambda consumers** — add in reliability/resilience section

### Out of Scope for This Design Document

- [ ] **IoT Greengrass edge processing** — adds significant complexity; devices described as simple telemetry senders; defer unless evaluator requests edge scenario
- [ ] **Real-time ML inference on telemetry stream** — beyond stated requirements; note as future extension
- [ ] **Multi-region active-active** — overkill for thousands of devices; single-region with cross-region S3 replication sufficient
- [ ] **AWS IoT FleetWise** — automotive-specific; not relevant for generic industrial telemetry

---

## Feature Prioritization Matrix

| Feature | Architecture Value | Implementation Complexity | Priority |
|---------|-------------------|--------------------------|----------|
| IoT Core MQTT ingestion | HIGH | LOW | P1 |
| IoT Rules Engine routing | HIGH | LOW | P1 |
| Device Shadow (disconnected command) | HIGH | MEDIUM | P1 |
| Amazon Timestream (telemetry storage) | HIGH | MEDIUM | P1 |
| S3 Data Lake + Glue ETL + Parquet | HIGH | HIGH | P1 |
| Alarm detection + SNS notification | HIGH | MEDIUM | P1 |
| REST API (API Gateway + Lambda) | HIGH | MEDIUM | P1 |
| Web visualization (Grafana or SPA) | HIGH | MEDIUM | P1 |
| VPC private database isolation | HIGH | LOW | P1 |
| X.509 cert auth + IAM least-privilege | HIGH | LOW | P1 |
| Technology comparison tables | HIGH | MEDIUM | P1 |
| Mermaid architecture diagrams | HIGH | MEDIUM | P1 |
| Kinesis Firehose ingestion buffer | MEDIUM | MEDIUM | P2 |
| EventBridge alarm fan-out | MEDIUM | LOW | P2 |
| Glue Data Catalog + Athena | MEDIUM | LOW | P2 |
| IoT Device Defender | MEDIUM | LOW | P2 |
| S3 Intelligent-Tiering | LOW | LOW | P2 |
| DLQ on Lambda consumers | MEDIUM | LOW | P2 |
| Cost analysis section | MEDIUM | HIGH | P2 |
| Azure/GCP equivalent (bonus) | LOW | HIGH | P3 |
| IoT Greengrass edge | LOW | HIGH | P3 |
| Multi-region HA | LOW | HIGH | P3 |

**Priority key:**
- P1: Must document — required by brief or evaluator will notice the gap
- P2: Should document — demonstrates architectural depth and AWS expertise
- P3: Defer — adds complexity disproportionate to assessment scope

---

## Competitor / Platform Feature Benchmark

How major IoT platforms handle these features, to inform design decisions:

| Feature | AWS IoT Core (native) | Azure IoT Hub | GCP Cloud IoT | Implication for Design |
|---------|----------------------|---------------|---------------|------------------------|
| Disconnected device commands | Device Shadow (desired/reported/delta) | Device Twin | Device config | Use Shadow; it is the AWS-native pattern |
| Telemetry routing | IoT Rules Engine (SQL) | Event Grid + Stream Analytics | Pub/Sub + Dataflow | Rules Engine is sufficient for this scale |
| Time-series storage | Amazon Timestream | Azure Time Series Insights | BigTable + BigQuery | Timestream is the clear AWS choice |
| Data Lake format | S3 + Glue + Parquet | ADLS Gen2 + Synapse | GCS + BigQuery | S3+Parquet is well-established, AI-ready |
| Visualization | Managed Grafana, QuickSight | Power BI | Looker | Grafana preferred for operational telemetry |
| Security | X.509 + IoT Policies + Defender | SAS tokens + Azure Defender | Google-managed certs | X.509 per device is stronger than SAS tokens |

---

## Sources

- [AWS IoT Core Documentation — Device Shadow Service](https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html) — HIGH confidence (official docs)
- [7 Patterns for IoT Data Ingestion and Visualization — AWS IoT Blog](https://aws.amazon.com/blogs/iot/7-patterns-for-iot-data-ingestion-and-visualization-how-to-decide-what-works-best-for-your-use-case/) — HIGH confidence (official AWS blog)
- [AWS IoT Security Best Practices](https://docs.aws.amazon.com/iot/latest/developerguide/security-best-practices.html) — HIGH confidence (official docs)
- [Build a Data Lake Foundation with AWS Glue and Amazon S3](https://aws.amazon.com/blogs/big-data/build-a-data-lake-foundation-with-aws-glue-and-amazon-s3/) — HIGH confidence (official AWS blog)
- [Monitor IoT Device Health at Scale with Amazon Managed Grafana](https://aws.amazon.com/blogs/mt/monitor-iot-device-health-at-scale-with-amazon-managed-grafana/) — HIGH confidence (official AWS blog)
- [Three Cost-Effective Design Patterns for AWS IoT Data Ingestion — Trek10](https://www.trek10.com/blog/three-cost-effective-design-patterns-for-aws-iot-data-ingestion) — MEDIUM confidence (verified technical blog, cost figures from SQS vs Kinesis comparison)
- [Amazon Timestream vs DynamoDB — Superluminar lessons learned (2025)](https://superluminar.io/2025/03/21/an-overview-and-lessons-learned-from-storing-and-analysing-iot-data-in-aws-timestream/) — MEDIUM confidence (practitioner experience, 2025)
- [Building Event-Driven Architectures with IoT Sensor Data — AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/building-event-driven-architectures-with-iot-sensor-data/) — HIGH confidence (official AWS Architecture blog)
- [Common Architecture Patterns to Securely Connect IoT Devices — AWS IoT Blog](https://aws.amazon.com/blogs/iot/common-architecture-patterns-to-securely-connect-iot-devices-to-aws-using-private-networks/) — HIGH confidence (official AWS blog)
- [AWS IoT Architecture Center](https://aws.amazon.com/architecture/iot/) — HIGH confidence (official reference architectures)

---

*Feature research for: AWS IoT monitoring platform (architecture design document)*
*Researched: 2026-03-27*
