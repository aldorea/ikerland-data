# IoT Cloud Architecture — AWS

Architecture design documentation for an IoT device monitoring platform on AWS. This document set describes a serverless, decoupled architecture supporting thousands of devices with hourly telemetry, real-time alarm notifications, and a web-based operator dashboard.

## Architecture Documents

| # | Document | Layer | Description |
|---|----------|-------|-------------|
| 01 | [Security Foundation](01-security-foundation.md) | Cross-cutting | VPC topology, IAM roles, KMS encryption, TLS, WAF |
| 02 | [Device Connectivity & Ingestion](02-device-connectivity-ingestion.md) | IoT Ingestion | IoT Core, MQTT topics, Rules Engine, Basic Ingest |
| 03 | [Device Management](03-device-management.md) | Device Management | Device Shadow, Fleet Provisioning, Thing Types/Groups |
| 04 | [Data Pipeline Processing](04-data-pipeline-processing.md) | Processing | Kinesis Data Streams, Lambda batch consumer, DLQ |
| 05 | [Storage Layer](05-storage-layer.md) | Storage | Timestream, DynamoDB, Aurora Serverless v2, RDS Proxy |
| 06 | [Alarm Notifications](06-alarm-notifications.md) | Notifications | SNS fan-out, SES email, EventBridge extensibility, dedup |
| 07 | [Data Lake & ETL](07-data-lake-etl.md) | Data Lake | S3 medallion (Bronze/Silver/Gold), Glue ETL, Athena |
| 08 | [API Layer](08-api-layer.md) | REST API | API Gateway HTTP API v2, Cognito JWT auth, Lambda handlers |
| 09 | [Web Frontend](09-web-frontend.md) | Web Frontend | S3 + CloudFront OAC, SPA hosting, device command flow |
| 10 | [Architecture Overview & Sequences](10-overview-and-sequences.md) | Cross-cutting | System overview diagram, 3 end-to-end sequence diagrams |
| 11 | [Cost Analysis](11-cost-analysis.md) | Cross-cutting | Per-service cost estimates, optimization strategies |

## Reading Order

**For a quick overview:** Start with [10-overview-and-sequences.md](10-overview-and-sequences.md) for the full system diagram and key flows (telemetry ingestion, alarm notification, command delivery).

**For deep understanding:** Read documents 01 through 09 in order — each layer builds on the previous.

**For cost assessment:** See [11-cost-analysis.md](11-cost-analysis.md) for per-service pricing and optimization strategies ($40–$303/month range depending on device count).

## Key Design Principles

- **Serverless-first:** Zero idle cost for compute (Lambda, API Gateway, Athena). Pay only for usage.
- **Decoupled layers:** Each layer communicates via managed AWS services (Kinesis, SNS, S3). No direct service-to-service coupling.
- **Security by default:** All databases in private VPC subnets. No public endpoints. KMS encryption at rest. TLS in transit.
- **Cost-optimized:** IoT Core Basic Ingest, Graviton2 Lambda, Parquet partitioning, DynamoDB TTL — estimated $40–$145/month for thousands of devices at modest load.
- **Disconnected-device-ready:** Device Shadow desired/delta pattern handles normally-disconnected devices connecting only hourly. Commands are queued and delivered on reconnect.
