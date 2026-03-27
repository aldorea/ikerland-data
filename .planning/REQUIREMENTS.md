# Requirements: IoT Cloud Architecture Design (AWS)

**Defined:** 2026-03-27
**Core Value:** A well-documented, decoupled, scalable, and cost-effective AWS architecture with justified design decisions

## v1 Requirements

Requirements for the architecture design document deliverable.

### Ingestion

- [ ] **INGT-01**: Architecture documents IoT Core as the MQTT/HTTPS entry point for device telemetry, config, and alarm events in JSON format
- [ ] **INGT-02**: Architecture documents IoT Rules Engine for message routing and filtering by message type (telemetry, config, alarm)
- [ ] **INGT-03**: Architecture documents Kinesis Data Firehose as ingestion buffer for cost-optimized telemetry processing at scale
- [ ] **INGT-04**: Architecture includes topic namespace design (e.g., `devices/{thingName}/telemetry`, `/alarm`, `/config`)

### Device Management

- [ ] **DEVM-01**: Architecture documents Device Shadow for command/config delivery to normally-disconnected devices (desired/reported/delta flow)
- [ ] **DEVM-02**: Architecture includes sequence diagram showing full command delivery lifecycle (web → API → Shadow → device reconnect → delta apply → reported update)
- [ ] **DEVM-03**: Architecture documents X.509 per-device certificate authentication with IoT policy variables (`${iot:ThingName}`)
- [ ] **DEVM-04**: Architecture documents device fleet grouping via Thing Types and Thing Groups

### Processing

- [ ] **PROC-01**: Architecture documents Lambda-based message processing with type-dependent routing
- [ ] **PROC-02**: Architecture documents SQS dead-letter queues for failed processing (no silent data loss)
- [ ] **PROC-03**: Architecture documents batched processing via Kinesis (not per-message Lambda invocation) for telemetry hot path

### Storage

- [ ] **STOR-01**: Architecture documents Amazon Timestream for time-series telemetry (hot storage with memory + magnetic tiers)
- [ ] **STOR-02**: Architecture documents DynamoDB for device metadata, state, and latest-value cache
- [ ] **STOR-03**: Architecture documents Aurora Serverless v2 for user/role/alert-rule relational data
- [ ] **STOR-04**: Architecture includes comparison table of storage alternatives with pros/cons and recommendation

### Alarm & Notifications

- [ ] **ALRM-01**: Architecture documents alarm detection via IoT Rules Engine threshold evaluation
- [ ] **ALRM-02**: Architecture documents SNS fan-out for email notifications
- [ ] **ALRM-03**: Architecture documents EventBridge for extensible multi-channel notification routing (email, SMS, webhook, future channels)
- [ ] **ALRM-04**: Architecture documents alarm deduplication strategy (prevent notification storms)

### API

- [ ] **API-01**: Architecture documents API Gateway (HTTP API) + Lambda for REST API
- [ ] **API-02**: Architecture documents Cognito for user authentication and JWT-based authorization
- [ ] **API-03**: Architecture documents API deployed within the AWS platform (not external)

### Web Application

- [ ] **WEB-01**: Architecture documents S3 + CloudFront for SPA hosting (or Managed Grafana alternative with comparison)
- [ ] **WEB-02**: Architecture documents web-to-device command flow (UI → API → Device Shadow)
- [ ] **WEB-03**: Architecture documents CloudFront OAC for secure S3 origin access

### Data Lake & ETL

- [ ] **LAKE-01**: Architecture documents S3 Data Lake with medallion pattern (Bronze/Silver/Gold zones)
- [ ] **LAKE-02**: Architecture documents AWS Glue ETL for JSON → Parquet transformation
- [ ] **LAKE-03**: Architecture documents periodic/event-driven ETL trigger from database to Data Lake
- [ ] **LAKE-04**: Architecture documents Athena for ad-hoc SQL queries on Parquet data
- [ ] **LAKE-05**: Architecture documents Hive-style partitioning by device and date for query cost optimization

### Security & Networking

- [ ] **SEC-01**: Architecture documents VPC topology with private subnets for all databases (no public access)
- [ ] **SEC-02**: Architecture documents VPC endpoints (Gateway + Interface) for DynamoDB, S3, Timestream, etc.
- [ ] **SEC-03**: Architecture documents IAM least-privilege roles for all service-to-service communication
- [ ] **SEC-04**: Architecture documents KMS encryption at rest for S3, DynamoDB, Timestream
- [ ] **SEC-05**: Architecture documents TLS 1.2+ encryption in transit (enforced by IoT Core)
- [ ] **SEC-06**: Architecture documents WAF on CloudFront and API Gateway

### Documentation Quality

- [ ] **DOC-01**: Architecture includes Mermaid diagram of the complete system (all layers and data flows)
- [ ] **DOC-02**: Architecture includes per-layer Mermaid diagrams with component detail
- [ ] **DOC-03**: Architecture includes technology comparison tables for each major decision (alternatives, pros, cons, recommendation)
- [ ] **DOC-04**: Architecture includes cost optimization analysis with estimated monthly cost range
- [ ] **DOC-05**: Architecture includes sequence diagrams for key flows (telemetry ingestion, alarm notification, command delivery)

## v2 Requirements

Deferred — optional bonus items not in primary scope.

### Multi-Cloud

- **MCLD-01**: Equivalent architecture on Azure (IoT Hub, Event Grid, Cosmos DB, etc.)
- **MCLD-02**: Equivalent architecture on GCP (Cloud IoT Core, Pub/Sub, BigQuery, etc.)
- **MCLD-03**: Kubernetes-based alternative (EMQX, Kafka, TimescaleDB, etc.)

### Advanced Features

- **ADV-01**: AWS IoT Device Defender for anomaly detection
- **ADV-02**: AWS IoT Greengrass for edge processing
- **ADV-03**: Amazon Managed Grafana as primary visualization (vs custom SPA)
- **ADV-04**: CI/CD pipeline design for infrastructure deployment
- **ADV-05**: Disaster recovery and multi-region design

## Out of Scope

| Feature | Reason |
|---------|--------|
| Infrastructure as Code (Terraform/CDK/CFN) | Design exercise, not implementation |
| Actual AWS resource provisioning | No accounts or deployments |
| Application source code | Focus is architecture, not coding |
| Detailed pricing calculator output | High-level cost principles only |
| Real-time streaming (sub-second) | Devices connect hourly — overkill |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|
| INGT-01 | Pending | Pending |
| INGT-02 | Pending | Pending |
| INGT-03 | Pending | Pending |
| INGT-04 | Pending | Pending |
| DEVM-01 | Pending | Pending |
| DEVM-02 | Pending | Pending |
| DEVM-03 | Pending | Pending |
| DEVM-04 | Pending | Pending |
| PROC-01 | Pending | Pending |
| PROC-02 | Pending | Pending |
| PROC-03 | Pending | Pending |
| STOR-01 | Pending | Pending |
| STOR-02 | Pending | Pending |
| STOR-03 | Pending | Pending |
| STOR-04 | Pending | Pending |
| ALRM-01 | Pending | Pending |
| ALRM-02 | Pending | Pending |
| ALRM-03 | Pending | Pending |
| ALRM-04 | Pending | Pending |
| API-01 | Pending | Pending |
| API-02 | Pending | Pending |
| API-03 | Pending | Pending |
| WEB-01 | Pending | Pending |
| WEB-02 | Pending | Pending |
| WEB-03 | Pending | Pending |
| LAKE-01 | Pending | Pending |
| LAKE-02 | Pending | Pending |
| LAKE-03 | Pending | Pending |
| LAKE-04 | Pending | Pending |
| LAKE-05 | Pending | Pending |
| SEC-01 | Pending | Pending |
| SEC-02 | Pending | Pending |
| SEC-03 | Pending | Pending |
| SEC-04 | Pending | Pending |
| SEC-05 | Pending | Pending |
| SEC-06 | Pending | Pending |
| DOC-01 | Pending | Pending |
| DOC-02 | Pending | Pending |
| DOC-03 | Pending | Pending |
| DOC-04 | Pending | Pending |
| DOC-05 | Pending | Pending |

**Coverage:**
- v1 requirements: 41 total
- Mapped to phases: 0
- Unmapped: 41 ⚠️

---
*Requirements defined: 2026-03-27*
*Last updated: 2026-03-27 after initial definition*
