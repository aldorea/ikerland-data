## Architecture Overview

This document provides the 30,000-foot view of the IoT monitoring platform architecture, followed by end-to-end sequence diagrams for the three key operational flows. Each component is documented in detail in its per-layer document (01–09). This overview shows how all layers connect.

---

### System Architecture Overview

```mermaid
flowchart LR
    subgraph DEVICES["IoT Devices"]
        DEV["Device Fleet\n1K-10K devices"]
    end

    subgraph INGESTION["Layer 1: IoT Ingestion"]
        IOT["AWS IoT Core\nMQTT/HTTPS\nRules Engine"]
        SHADOW["Device Shadow"]
    end

    subgraph PROCESSING["Layer 2: Message Processing"]
        KDS["Kinesis\nData Streams"]
        KDF["Kinesis\nData Firehose"]
        LAM_PROC["Lambda\nProcessors"]
    end

    subgraph STORAGE["Layer 3: Storage"]
        TS[("Timestream\nTime-Series")]
        DDB[("DynamoDB\nDevice State")]
        AUR[("Aurora\nPostgreSQL\nServerless v2")]
    end

    subgraph DATALAKE["Layer 4: Data Lake"]
        S3[("S3 Data Lake\nBronze/Silver/Gold")]
        GLUE["AWS Glue\nETL"]
        ATHENA["Amazon Athena\nSQL Queries"]
    end

    subgraph NOTIFICATIONS["Layer 5: Notifications"]
        SNS["Amazon SNS\nFan-out"]
        SES["Amazon SES\nEmail"]
        EB["EventBridge\nExtensibility"]
    end

    subgraph API["Layer 6: REST API"]
        APIGW["API Gateway\nHTTP API v2"]
        LAM_API["Lambda\nAPI Handlers"]
        PROXY["RDS Proxy"]
    end

    subgraph FRONTEND["Layer 7: Web Frontend"]
        CF["CloudFront\n+ WAF"]
        S3SPA["S3 SPA"]
        COG["Cognito\nUser Pools"]
    end

    DEV -->|"MQTT telemetry"| IOT
    IOT -->|"Rules Engine"| KDS
    IOT -->|"Rules Engine"| KDF
    IOT -->|"alarm events"| LAM_PROC
    KDS --> LAM_PROC
    LAM_PROC --> TS
    LAM_PROC --> DDB
    LAM_PROC -->|"alarm"| SNS
    SNS --> SES
    SNS --> EB
    KDF --> S3
    S3 -->|"Bronze/Silver/Gold"| GLUE
    GLUE -->|"Parquet"| S3
    S3 --> ATHENA
    CF -->|"static assets"| S3SPA
    CF -->|"/api/*"| APIGW
    COG -->|"JWT"| APIGW
    APIGW --> LAM_API
    LAM_API --> TS
    LAM_API --> DDB
    LAM_API --> PROXY --> AUR
    LAM_API -->|"commands"| SHADOW
    SHADOW -.->|"delta on reconnect"| DEV
```

#### Per-Layer Document Reference

| Layer | Per-Layer Document | Key Components |
|-------|-------------------|----------------|
| Security Foundation | [01-security-foundation.md](01-security-foundation.md) | VPC, IAM, KMS, WAF |
| IoT Ingestion | [02-device-connectivity-ingestion.md](02-device-connectivity-ingestion.md) | IoT Core, Rules Engine, Basic Ingest |
| Device Management | [03-device-management.md](03-device-management.md) | Device Shadow, Fleet Provisioning |
| Message Processing | [04-data-pipeline-processing.md](04-data-pipeline-processing.md) | Kinesis Data Streams, Lambda batch consumer |
| Storage | [05-storage-layer.md](05-storage-layer.md) | Timestream, DynamoDB, Aurora Serverless v2 |
| Alarm Notifications | [06-alarm-notifications.md](06-alarm-notifications.md) | SNS, SES, EventBridge, deduplication |
| Data Lake & ETL | [07-data-lake-etl.md](07-data-lake-etl.md) | S3 Medallion, Glue ETL, Athena |
| REST API | [08-api-layer.md](08-api-layer.md) | API Gateway HTTP API v2, Cognito, Lambda |
| Web Frontend | [09-web-frontend.md](09-web-frontend.md) | CloudFront, S3 OAC, SPA |

---

### Telemetry Ingestion Flow

This sequence covers the end-to-end path of a telemetry message from device publish to storage in both the operational hot store (Timestream) and the analytical cold store (S3 Data Lake).

```mermaid
sequenceDiagram
    participant DEV as Device
    participant IOT as IoT Core
    participant RE as Rules Engine
    participant KDS as Kinesis Data Streams
    participant KDF as Kinesis Data Firehose
    participant LAM as Lambda (telemetry-processor)
    participant TS as Timestream
    participant DDB as DynamoDB
    participant S3B as S3 Bronze

    DEV->>IOT: MQTT PUBLISH devices/{thingName}/telemetry (JSON, 1KB)
    IOT->>RE: Trigger TelemetryRule (SELECT * FROM 'devices/+/telemetry')
    RE->>KDS: Fan-out - hot path record
    RE->>KDF: Fan-out - cold path record
    Note over KDS: Buffering batch (size 100-500)
    KDS->>LAM: Invoke with batch (Event Source Mapping)
    LAM->>TS: Write time-series record (thingName, ts, values)
    LAM->>DDB: Update latest-value cache (PK=deviceId, SK=latest)
    KDF->>S3B: Buffer + deliver raw JSON to bronze/year=/month=/day=/
    Note over S3B: Glue ETL transforms Bronze to Silver (Parquet) to Gold (aggregated) on schedule
```

> See [02-device-connectivity-ingestion.md](02-device-connectivity-ingestion.md) for IoT Core and Rules Engine configuration.
> See [04-data-pipeline-processing.md](04-data-pipeline-processing.md) for Kinesis and Lambda batch processing.
> See [05-storage-layer.md](05-storage-layer.md) for Timestream and DynamoDB storage design.
> See [07-data-lake-etl.md](07-data-lake-etl.md) for the Data Lake cold path (Glue ETL and Athena).

---

### Alarm Notification Flow

This sequence covers the path from a device reporting a threshold breach to an email notification reaching the operator, including deduplication to prevent notification storms.

```mermaid
sequenceDiagram
    participant DEV as Device
    participant IOT as IoT Core
    participant RE as Rules Engine
    participant LAME as Lambda (alarm-evaluator)
    participant DDB as DynamoDB (dedup)
    participant AUR as Aurora (RDS Proxy)
    participant SNS as Amazon SNS
    participant LAMS as Lambda (ses-sender)
    participant SES as Amazon SES
    participant EB as EventBridge

    DEV->>IOT: MQTT PUBLISH devices/{thingName}/alarm (threshold breach)
    IOT->>RE: Trigger AlarmRule (WHERE value exceeds threshold)
    RE->>LAME: Invoke Lambda (alarm-evaluator) asynchronously
    LAME->>AUR: Query alert rules and device group context (via RDS Proxy)
    LAME->>DDB: Conditional write (attribute_not_exists, 15-min TTL)
    alt New alarm (conditional write succeeds)
        LAME->>SNS: Publish to alarm-notifications topic
        SNS->>LAMS: Fan-out to Lambda (ses-sender) subscription
        SNS->>EB: Fan-out to iot-alarm-bus (extensibility)
        LAMS->>SES: Send formatted alarm email
    else Duplicate within 15-min window (conditional write fails)
        LAME->>LAME: Log suppressed duplicate, exit
    end
    Note over LAME: On Lambda failure, DestinationConfig.OnFailure routes to SQS DLQ
```

> See [02-device-connectivity-ingestion.md](02-device-connectivity-ingestion.md) for IoT Rules Engine.
> See [06-alarm-notifications.md](06-alarm-notifications.md) for the full alarm pipeline, deduplication strategy, and EventBridge extensibility.

---

### Command Delivery Flow

This sequence covers the full end-to-end path of an operator sending a command from the web dashboard to a normally-disconnected device. The command is queued in IoT Device Shadow and delivered when the device next connects.

```mermaid
sequenceDiagram
    participant BROWSER as Browser (Operator)
    participant COG as Cognito User Pools
    participant APIGW as API Gateway
    participant LAM as Lambda (command-handler)
    participant SHADOW as IoT Device Shadow
    participant DDB as DynamoDB (audit)
    participant DEV as Device

    Note over BROWSER: Operator already authenticated via Cognito (see 08-api-layer.md auth flow)
    BROWSER->>APIGW: POST /devices/{thingName}/commands + JWT header
    APIGW->>COG: JWT authorizer validates token (issuer, audience, expiry)
    COG-->>APIGW: Token valid, claims extracted
    APIGW->>LAM: Invoke Lambda (command-handler)
    LAM->>SHADOW: UpdateThingShadow - set desired state
    SHADOW-->>LAM: Shadow version returned
    LAM->>DDB: Audit log entry (commandId, thingName, operator, timestamp, shadowVersion)
    LAM-->>APIGW: 202 Accepted (command queued, not yet executed)
    APIGW-->>BROWSER: 202 Accepted + commandId for async status polling
    Note over DEV: Device reconnects hourly
    DEV->>SHADOW: Subscribe to delta topic ($aws/things/{thingName}/shadow/update/delta)
    SHADOW-->>DEV: Delta document (desired state differs from reported)
    DEV->>DEV: Apply command (reboot, config update, etc.)
    DEV->>SHADOW: UpdateThingShadow - set reported state = desired state
    SHADOW-->>DEV: Delta cleared (desired == reported)
```

> See [08-api-layer.md](08-api-layer.md) for Cognito authentication and API Gateway configuration.
> See [03-device-management.md](03-device-management.md) for the Device Shadow delta mechanism and shadow document structure.
> See [09-web-frontend.md](09-web-frontend.md) for the SPA hosting architecture and operator dashboard.

---

### Technology Decision Summary

This table provides a single-page view of all major technology decisions across the architecture. Each comparison table is located in its per-layer document with full pros/cons analysis.

| Decision | Alternatives Compared | Recommendation | Document |
|----------|-----------------------|----------------|----------|
| IoT Entry Point | IoT Core vs SiteWise vs self-managed MQTT | AWS IoT Core | [02-device-connectivity-ingestion.md](02-device-connectivity-ingestion.md) |
| Stream Buffer | Kinesis Data Streams vs MSK vs SQS | Kinesis Data Streams | [04-data-pipeline-processing.md](04-data-pipeline-processing.md) |
| Time-Series Store | Timestream vs DynamoDB-TS vs InfluxDB | Amazon Timestream | [05-storage-layer.md](05-storage-layer.md) |
| Relational Store | Aurora Serverless v2 vs RDS Provisioned vs DynamoDB-only | Aurora PostgreSQL Serverless v2 | [05-storage-layer.md](05-storage-layer.md) |
| ETL Trigger | EventBridge Scheduler vs S3-event-driven vs Glue Workflow | EventBridge Scheduler | [07-data-lake-etl.md](07-data-lake-etl.md) |
| Query Engine | Athena vs Redshift Spectrum vs EMR | Amazon Athena | [07-data-lake-etl.md](07-data-lake-etl.md) |
| API Front Door | HTTP API v2 vs REST API v1 vs App Runner | API Gateway HTTP API v2 | [08-api-layer.md](08-api-layer.md) |
| Auth Provider | Cognito User Pools vs IAM Identity Center vs self-managed | Amazon Cognito User Pools | [08-api-layer.md](08-api-layer.md) |
| Web Hosting | S3+CloudFront vs Managed Grafana vs Amplify Hosting | S3 + CloudFront | [09-web-frontend.md](09-web-frontend.md) |

> Each comparison table is located in its per-layer document with full pros/cons analysis. This summary provides a single-page view for evaluators. Per D-15, tables are referenced here, not duplicated.
