# Stack Research

**Domain:** AWS IoT Device Monitoring Platform
**Researched:** 2026-03-27
**Confidence:** HIGH (all major claims verified against official AWS docs and multiple sources)

---

## Recommended Stack

### Layer 1 — IoT Ingestion

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| AWS IoT Core | Current (managed) | Device connectivity, MQTT broker, auth, TLS termination | The standard AWS entry point for IoT. Handles certificates, policies, and bi-directional MQTT. Built-in Rules Engine routes messages to 20+ AWS targets without code. Native integration with Device Shadow for disconnected-device patterns. |
| AWS IoT Core — Basic Ingest | Reserved topic `$aws/rules/...` | High-volume telemetry bypass | Skips the pub/sub broker for device-to-cloud telemetry, reducing cost by ~50% on message pricing. Mandatory for thousands of devices sending hourly data at scale. |
| AWS IoT Core — Device Shadow | Named shadows per device | Command queuing for disconnected devices | Persists `desired`/`reported`/`delta` state in cloud. When a device reconnects hourly, it automatically receives pending commands via the delta topic. This is the canonical AWS pattern for normally-disconnected devices. |
| AWS IoT Core — Rules Engine | SQL-based rules | Message routing and conditional dispatch | Built into IoT Core at no extra charge per rule (only per action). Routes telemetry to Kinesis, alarm events to SNS/Lambda, config messages to DynamoDB. Eliminates a separate router service. |

### Layer 2 — Message Processing Pipeline

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| Amazon Kinesis Data Streams | On-Demand or Provisioned | Real-time stream buffer between IoT Core and processors | Decouples ingestion from processing. Stores data 24h–7d for replay. Scales to millions of records/sec. IoT Rules Engine can fan-out to Kinesis without Lambda overhead. |
| Amazon Kinesis Data Firehose | Serverless | Direct delivery to S3 Data Lake (raw landing zone) | Fully managed, zero ops. Buffers and batches records, converts to Parquet via built-in Glue catalog integration, and partitions by date. Cheapest path for telemetry → S3. Zero Buffering (Dec 2023) delivers within ~5 seconds. |
| AWS Lambda | Python 3.12 / Node.js 20 | Stateless message transformation, alarm evaluation, complex routing | Invoked per-batch from Kinesis Data Streams. Handles: format normalization, threshold checks, enrichment with device metadata from DynamoDB. Stateless — scales automatically. Use batch size 100–500 to amortize cold starts. |

### Layer 3 — Storage

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| Amazon Timestream for LiveAnalytics | Serverless | Hot/warm time-series telemetry storage | Purpose-built for IoT metrics. Up to 1,000× faster and 1/10th the cost of relational DBs for time-range queries. Automatic tiering: in-memory (hours) → magnetic (months). Native integration with Grafana and QuickSight. Scales to millions of writes/sec. |
| Amazon DynamoDB | On-Demand | Device metadata, shadows state, command queue, device registry | Low-latency key-value lookups for device-centric data (device info, last seen, config, pending commands). On-Demand billing eliminates capacity planning. VPC endpoint enforces private-only access. |
| Amazon S3 (Data Lake) | Standard + Intelligent-Tiering | Raw JSON landing zone and Parquet Data Lake | Virtually unlimited, 99.999999999% durable, cheapest long-term store. Organized in medallion architecture: `bronze/` (raw JSON), `silver/` (cleaned Parquet), `gold/` (aggregated Parquet). Serves as the source of truth for historical analytics. |
| Amazon RDS (Aurora PostgreSQL Serverless v2) | Serverless v2 | Relational data: users, roles, alert rules, device groups | Aurora Serverless v2 scales to zero when idle (cost-critical for this design exercise). PostgreSQL dialect is standard. Deployed in private VPC subnets — no public endpoint. Alternative to standalone PostgreSQL with managed HA. |

### Layer 4 — Data Lake / ETL

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| AWS Glue | Serverless ETL | Bronze → Silver → Gold transformations, Parquet conversion | Serverless Spark-based ETL. Scheduled jobs run periodically (hourly or daily) to transform raw JSON → Parquet with partitioning. Glue Data Catalog auto-discovers schema and registers tables for Athena. |
| AWS Glue Data Catalog | Managed | Central metadata registry for S3 data lake | Single source of schema truth for Athena, Glue jobs, and Lake Formation. Free up to 1M objects/month. Required for Firehose Parquet conversion. |
| AWS Lake Formation | Managed | Fine-grained access control for Data Lake | Governs who can query which partitions/columns in S3. Applies column-level security and row filters. Recommended over raw S3 bucket policies for multi-team access. |
| Amazon Athena | Serverless SQL | Ad-hoc queries on S3 Parquet Data Lake | Pay-per-query ($5/TB scanned). No infrastructure. Integrates with Glue Catalog. Parquet with partitioning reduces scan costs by 80–90%. Used by analysts and for API-backed dashboards on historical data. |

### Layer 5 — REST API

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| Amazon API Gateway | HTTP API (v2) | REST API front door for web application | HTTP API is 70% cheaper than REST API v1 and has lower latency. Supports JWT authorizer directly (no Lambda authorizer needed for basic auth). Integrates natively with Lambda, VPC Link for private targets. |
| AWS Lambda | Python 3.12 / Node.js 20 | API business logic handlers | Serverless handlers per endpoint group. No idle cost. Auto-scales. For this scale (thousands of devices, N concurrent users), Lambda is cost-optimal vs a container always running. |
| Amazon Cognito User Pools | Managed | Web app user authentication and JWT issuance | Standard CIAM for customer-facing apps. Issues JWTs consumed by API Gateway HTTP API authorizer. Supports MFA, social login (optional). Direct IoT Core integration via Cognito Identity Pools for federated device/user access. |

### Layer 6 — Web Hosting

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| Amazon S3 | Standard | Static web asset storage (React/Vue/Angular SPA) | Zero-cost hosting for static files. Versioned deployments. Not publicly accessible directly — served exclusively through CloudFront. |
| Amazon CloudFront | Managed CDN | HTTPS CDN for SPA, global edge caching | Mandatory for production SPAs: HTTPS enforcement, custom domain + ACM certificate, cache policies, edge security headers. Integrates with WAF. Cost: $0.0085–$0.012/GB transferred. Free HTTPS with ACM. |
| AWS Certificate Manager (ACM) | Free for public certs | TLS certificate provisioning | Free managed TLS certificates auto-renewed. Attach to CloudFront and API Gateway. Zero cert management overhead. |

### Layer 7 — Notifications

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| Amazon SNS | Standard topics | Fan-out alarm notifications, multi-channel dispatch | Native IoT Rules Engine action target. One SNS topic can fan out to: email (SES), SMS, HTTP webhooks, Lambda, SQS. Standard pattern for IoT alarms. Pay-per-message ($0.50/1M). |
| Amazon SES | Managed email | Email notifications for alarms and reports | Cheapest bulk email service on AWS ($0.10/1K emails). Handles email templates, bounces, and compliance. Use for alarm emails and periodic reports. |
| Amazon EventBridge | Serverless events | Rule-based alarm orchestration with filtering | For complex alarm rules (rate-of-change, multi-signal correlation), EventBridge rules with pattern matching are cleaner than embedded Lambda logic. Also enables future integrations (PagerDuty, Slack via EventBridge Pipes). |

### Layer 8 — Security

| Service | Version/Tier | Purpose | Why Recommended |
|---------|--------------|---------|-----------------|
| AWS IAM | Managed | Service-to-service permissions, least-privilege policies | Foundation of all AWS security. Every Lambda, Glue job, and IoT Rule action uses an IAM role with minimum required permissions. Never use root credentials. |
| AWS IoT Core Policies | MQTT-level ACLs | Per-device MQTT topic authorization | Scoped to individual device certificates. Prevent device A from publishing to device B's topic. Use substitution templates (`${iot:ClientId}`) for scalable per-device policies. |
| Amazon VPC | Managed network | Private network isolation for all data stores | RDS Aurora, DynamoDB (via VPC endpoint), and S3 (via VPC gateway endpoint) are accessible only from within the VPC. Lambda functions processing data run in VPC private subnets. No database has a public endpoint. |
| VPC Endpoints (Gateway + Interface) | Managed | Private routing for S3, DynamoDB, Kinesis, Secrets Manager | Eliminates internet traversal for internal AWS service calls. Required for: S3 (Gateway endpoint, free), DynamoDB (Gateway endpoint, free), Kinesis/SQS/Secrets Manager (Interface endpoints, ~$7/month each). |
| AWS Secrets Manager | Managed | Database credentials, API keys rotation | Automatic credential rotation for RDS Aurora. Lambda functions retrieve secrets at startup via SDK — no hardcoded credentials. Costs $0.40/secret/month. |
| AWS KMS | Managed | Encryption key management | Customer-managed keys (CMK) for: S3 server-side encryption, DynamoDB encryption at rest, RDS storage encryption, Kinesis stream encryption. Provides audit trail via CloudTrail. |
| AWS CloudTrail | Managed | API audit logging | Logs all AWS API calls. Mandatory for security compliance. Detects unauthorized access. Store logs in S3 with KMS encryption. Free for management events; $2/100K data events. |
| AWS WAF | Managed | Web application firewall for API Gateway and CloudFront | Protects against SQL injection, XSS, rate limiting, geo-blocking. Attach to CloudFront distribution and API Gateway stage. Cost: $5/month per WebACL + $1/million requests. |
| Amazon Cognito Identity Pools | Managed | Federated credentials for web app → AWS services | Grants authenticated web users temporary IAM credentials scoped to their permissions, enabling direct Timestream or S3 queries from the frontend without proxying through Lambda. |

### Supporting Tools (Design/Documentation Exercise)

| Tool | Purpose | Notes |
|------|---------|-------|
| AWS Well-Architected Framework — IoT Lens | Architecture review checklist | Validates design against AWS best practices for IoT |
| Amazon CloudWatch | Metrics, logs, alarms for all AWS services | Built-in observability. IoT Core, Lambda, API Gateway publish metrics automatically. |
| AWS X-Ray | Distributed tracing across Lambda, API Gateway | Identifies latency bottlenecks in the processing pipeline. |

---

## Alternatives Considered

### IoT Ingestion

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| IoT entry point | AWS IoT Core | AWS IoT SiteWise | SiteWise adds industrial asset modeling (OEE, production lines). Use it if requirements include hierarchical asset models or OPC-UA/SCADA integration. For general JSON telemetry with custom schemas, IoT Core + custom processing is more flexible. |
| IoT entry point | AWS IoT Core | Self-managed MQTT broker (EMQX on EC2/EKS) | EMQX can cut messaging costs 80% at very high volume (millions of messages/hour). Avoid for this design: adds operational complexity, loses Rules Engine integration, requires DevOps expertise. Only consider at extreme scale (10M+ messages/day). |
| Disconnected command delivery | Device Shadow | SQS queue per device | SQS is valid but requires custom polling logic on the device side. Device Shadow is native to IoT Core, auto-syncs on reconnect, and persists delta state — less code, same result. Use SQS if you need ordered command queues with FIFO guarantees. |

### Message Processing

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| Stream buffer | Kinesis Data Streams | Amazon MSK (Managed Kafka) | MSK is better if: migrating from on-premise Kafka, needing Kafka ecosystem (Kafka Connect, Kafka Streams), or processing >10M messages/day where MSK cost/feature ratio improves. For greenfield AWS-native IoT with thousands of devices, Kinesis is simpler, cheaper, and fully managed. |
| Stream buffer | Kinesis Data Streams | Amazon SQS | SQS lacks ordered processing guarantees (FIFO adds cost) and has no replay capability. Use SQS for command delivery queues (where per-device ordering matters less) but not for the main telemetry pipeline. |
| Batch delivery to S3 | Kinesis Data Firehose | Lambda writing directly to S3 | Direct Lambda → S3 writes add complexity (batching logic, retry handling, partitioning). Firehose handles all of this natively with built-in Parquet conversion. Only avoid Firehose if you need sub-second delivery — its minimum buffer is ~5 seconds (Zero Buffering feature). |
| Message processor | Lambda | AWS Fargate (ECS) | Fargate suits long-running, stateful workloads or WebSocket connections. For stateless per-message transformations, Lambda's per-invocation billing and zero idle cost wins. Switch to Fargate if processing logic exceeds Lambda's 15-minute timeout or 10GB memory limit. |

### Storage

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| Time-series telemetry | Amazon Timestream | Amazon DynamoDB (with time-sorted SK) | DynamoDB can store time-series but requires careful schema design (partition key = deviceId, sort key = timestamp). Time-range queries are less efficient than Timestream. Use DynamoDB for time-series if you need single-digit millisecond point lookups and already have DynamoDB expertise. Timestream is better for range scans, aggregations, and cost at scale. |
| Time-series telemetry | Amazon Timestream | InfluxDB on EC2 | InfluxDB has richer time-series query language (Flux). Avoid for AWS-first design: adds EC2 management, backup complexity, and loses native AWS integration. Not serverless. |
| Relational store | Aurora PostgreSQL Serverless v2 | Amazon RDS PostgreSQL (provisioned) | Provisioned RDS has lower per-hour cost at steady load but cannot scale to zero. For a design exercise, Serverless v2 is the right choice: zero idle cost, scales automatically, still uses Aurora's storage layer. Switch to provisioned RDS for production workloads with predictable 24/7 load. |
| Relational store | Aurora PostgreSQL Serverless v2 | Amazon DynamoDB (for everything) | DynamoDB can replace RDS for device metadata but is less natural for relational data (user roles, alert rule configurations with joins). Use DynamoDB only if you want a fully NoSQL architecture and can denormalize aggressively. |
| Data Lake format | Parquet on S3 | Apache Iceberg on S3 | Iceberg adds ACID transactions, schema evolution, and time-travel queries. Recommended if: multiple concurrent writers, frequent schema changes, or regulatory need for data versioning. Overkill for initial IoT platform. Migrate to Iceberg if Data Lake matures. |

### API Layer

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| API front door | API Gateway HTTP API + Lambda | API Gateway REST API (v1) | REST API v1 supports request validation, usage plans, and API keys natively. HTTP API is 70% cheaper and sufficient for JWT auth + Lambda integrations. Use REST API v1 only if you need API keys for third-party billing, per-client throttling, or SDK generation. |
| API front door | API Gateway HTTP API + Lambda | AWS App Runner (containerized API) | App Runner charges ~$6–7/month base even at zero traffic. Lambda charges $0 at zero traffic. For an IoT platform where API traffic may be bursty or low during off-hours, Lambda is more cost-efficient. Use App Runner for APIs with consistent high-concurrency traffic where Lambda cold starts matter. |
| Authentication | Amazon Cognito User Pools | AWS IAM Identity Center | IAM Identity Center is for workforce/employee access to AWS accounts. Cognito User Pools is for application user authentication (external users, operators). They can be combined: Identity Center as SAML IdP for Cognito, if corporate SSO is needed. |

### Web Hosting

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| SPA hosting | S3 + CloudFront | AWS Amplify Hosting | Amplify Hosting wraps S3 + CloudFront with CI/CD, branch previews, and environment variables. Better for teams needing developer-friendly deployment pipelines. For architecture documentation purposes, S3 + CloudFront shows the underlying components explicitly — better for a technical assessment. In production, Amplify is a valid and often preferred choice. |
| SPA hosting | S3 + CloudFront | EC2 / ECS nginx | Avoid: adds server management, no global CDN by default, higher cost. Only warranted for server-side rendering (SSR) frameworks that can't be pre-rendered. |

### Notifications

| Category | Recommended | Alternative | Why Not / When to Use Instead |
|----------|-------------|-------------|-------------------------------|
| Alarm fan-out | SNS + SES | SNS email subscription only | SNS email subscriptions lack HTML templates and cannot be unsubscribed by users programmatically. Use SES for transactional emails (alarm notifications, reports) and SNS as the fan-out hub to SES Lambda targets. |
| Complex alarm logic | EventBridge | Step Functions | Step Functions adds durable state machines — useful when alarm acknowledgment flow requires human approval steps. EventBridge is lighter-weight for stateless rule evaluation. Use Step Functions if alarm workflow requires multi-step escalation (warn → escalate → page on-call). |

---

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Amazon Kinesis Data Analytics (legacy SQL) | Deprecated in favor of Managed Service for Apache Flink. No new features, end-of-life trajectory. | AWS Managed Service for Apache Flink (if complex stream analytics needed) or Lambda for simple transformations |
| Amazon IoT Analytics | Service is deprecated as of April 2023. AWS has stopped accepting new customers. | AWS Glue + S3 + Athena for IoT ETL and analytics |
| AWS Greengrass for this architecture | Greengrass is for edge compute (running logic on the device/gateway side). The requirement is cloud-side processing. | AWS IoT Core + Lambda for cloud-side processing |
| Amazon ElastiCache (Redis) as primary IoT store | Redis is expensive for IoT scale (instance-based). DynamoDB and Timestream serve the hot-data use cases at lower cost. | DynamoDB (metadata/state), Timestream (hot telemetry) |
| Self-managed Kafka on EC2 | Requires Zookeeper management, storage provisioning, manual scaling. No advantage over MSK or Kinesis for greenfield. | Amazon MSK (if Kafka ecosystem needed) or Kinesis Data Streams |
| RDS public endpoints | Violates the explicit security requirement: "database access restricted to internal AWS services only." | RDS in private VPC subnets with VPC endpoints and security groups |
| Amazon DynamoDB Streams as the primary message bus | DynamoDB Streams is for reacting to table changes, not for arbitrary message routing. Low throughput limits per shard. | Kinesis Data Streams for the primary message pipeline |
| AWS IoT Events for this use case | IoT Events is valuable for complex, stateful event detection (detector models). For basic threshold-breach alarms, it adds cost and complexity. | Lambda with threshold logic + SNS |

---

## Stack Patterns by Variant

**If device count scales beyond 100,000:**
- Replace Kinesis Data Streams On-Demand with Provisioned shards (predictable cost)
- Consider MSK Serverless instead of Kinesis (better throughput at sustained high volume)
- Add DynamoDB DAX in front of device metadata table (microsecond cache for frequently-read device configs)

**If real-time dashboard (sub-second latency) is required:**
- Add Amazon Managed Grafana connected directly to Timestream
- Or use AWS AppSync (GraphQL subscriptions over WebSocket) for live data push to SPA
- Kinesis Data Streams Enhanced Fan-Out (~70ms delivery) instead of standard polling

**If multi-region / disaster recovery is needed:**
- DynamoDB Global Tables for active-active replication
- S3 Cross-Region Replication for Data Lake redundancy
- IoT Core is already regional; use Route 53 latency routing to regional endpoints

**If budget is the primary constraint (MVP / prototype):**
- Use IoT Core Basic Ingest (cuts messaging cost ~50%)
- Replace Timestream with DynamoDB TTL-based time-series (acceptable for basic queries)
- Replace Aurora Serverless with DynamoDB (fully NoSQL, no relational DB cost)
- Use Lambda + S3 directly instead of Firehose (free at very low volume, below Firehose minimum charge)

**If corporate SSO integration is required:**
- Add IAM Identity Center as SAML 2.0 IdP for Cognito User Pool
- Users log in with existing corporate credentials (Active Directory / Okta)

---

## Cost Optimization Considerations

| Layer | Cost Driver | Optimization Strategy |
|-------|-------------|-----------------------|
| IoT Core | $1.00/million messages (5KB increments) | Use Basic Ingest for device-to-cloud telemetry (bypasses broker, ~50% cost reduction). Batch small sensor readings into one message per publish cycle. |
| Kinesis Data Streams | $0.015/shard-hour (provisioned) or pay-per-use (On-Demand) | Use On-Demand for unpredictable traffic. Switch to Provisioned when traffic pattern stabilizes (cheaper at steady load). |
| Kinesis Firehose | $0.029/GB ingested | Already cheap. Enable Parquet conversion to reduce S3 storage and Athena scan costs downstream. |
| Lambda | $0.0000002/request + $0.0000166667/GB-second | Use Graviton2 (arm64) runtime — 20% cheaper, 19% better performance. Optimize memory allocation (use AWS Lambda Power Tuning). Increase batch size from Kinesis to reduce invocation count. |
| Timestream | Writes: $0.50/million; Queries: $0.01/GB; Magnetic: $0.03/GB-month | Set magnetic store retention to match business requirement (e.g., 1 year). Older data auto-tiers from memory to magnetic. Archive to S3 via scheduled export for data older than retention period. |
| DynamoDB | On-Demand: $1.25/million write RCU, $0.25/million read RCU | Use On-Demand for variable IoT workloads. Enable TTL for transient data (command queue, session state) — deletions are free. |
| S3 Data Lake | $0.023/GB-month (Standard) | Use S3 Intelligent-Tiering for data older than 30 days (auto-moves cold data to cheaper tiers). Parquet + partitioning reduces Athena scan costs 80–90%. |
| Aurora Serverless v2 | $0.12/ACU-hour (scales to 0) | Scales to 0.5 ACU when idle (minimum billed). For a design exercise platform with light traffic, cost is negligible. For production, evaluate Reserved Instances once load is predictable. |
| API Gateway HTTP API | $1.00/million requests | 70% cheaper than REST API. No idle cost. |
| CloudFront | $0.0085/GB (first 10TB) | Enable compression (gzip/Brotli) for JS/CSS assets. Cache-Control headers maximize cache hit ratio. |
| CloudWatch | Free tier: 10 metrics, 3 dashboards | Use EMF (Embedded Metric Format) in Lambda for custom metrics — cheaper than PutMetricData API calls. |

**Estimated monthly cost range for thousands of devices (1,000–10,000), hourly telemetry:**
- IoT Core (1M messages/month): ~$1–$5
- Kinesis Data Streams (On-Demand, low volume): ~$5–$20
- Lambda processing: ~$1–$10
- Timestream (1M writes/month): ~$0.50–$5
- DynamoDB (On-Demand): ~$1–$10
- Aurora Serverless v2 (dev load): ~$20–$50/month
- S3 + Glue ETL: ~$5–$20
- API Gateway + Lambda (moderate traffic): ~$5–$20
- CloudFront (SPA delivery): ~$1–$5
- **Total estimate: $40–$145/month** for a thousands-of-devices monitoring platform at modest load

---

## Sources

- [AWS IoT Core MQTT Design Best Practices](https://docs.aws.amazon.com/whitepapers/latest/designing-mqtt-topics-aws-iot-core/mqtt-design-best-practices.html) — IoT Core ingestion patterns, Basic Ingest, topic design (HIGH confidence)
- [AWS IoT Device Shadow](https://docs.aws.amazon.com/iot/latest/developerguide/iot-device-shadows.html) — Disconnected device command delivery (HIGH confidence)
- [AWS IoT Rules Engine Actions](https://docs.aws.amazon.com/iot/latest/developerguide/iot-rule-actions.html) — Routing targets and Rule action pricing (HIGH confidence)
- [Choosing an AWS IoT service](https://docs.aws.amazon.com/decision-guides/latest/iot-on-aws-how-to-choose/iot.html) — IoT Core vs SiteWise vs other services (HIGH confidence)
- [Kinesis Data Streams vs Firehose comparison — Svix](https://www.svix.com/resources/faq/kinesis-data-stream-vs-firehose/) — Latency, retention, use case comparison (MEDIUM confidence)
- [Kinesis vs MSK — RisingWave](https://risingwave.com/blog/amazon-msk-and-kinesis-unraveling-the-key-differences/) — Decision framework for stream processing (MEDIUM confidence)
- [Amazon Timestream vs DynamoDB — Dynobase](https://dynobase.dev/dynamodb-vs-amazon-timestream/) — Time-series storage comparison (MEDIUM confidence)
- [Timestream IoT lessons learned — Superluminar (March 2025)](https://superluminar.io/2025/03/21/an-overview-and-lessons-learned-from-storing-and-analysing-iot-data-in-aws-timestream/) — Real-world Timestream experience (HIGH confidence, recent)
- [AWS Data Lake with S3, Glue, Lake Formation — Dev Community](https://dev.to/davidshaek/implementing-a-secure-data-governance-architecture-on-aws-with-s3-glue-athena-and-lake-formation-30gb) — Data Lake governance patterns (MEDIUM confidence)
- [IoT SNS Notifications — CloudThat](https://www.cloudthat.com/resources/blog/simple-iot-alert-notification-using-aws-iot-rule-and-sns) — IoT Rules Engine → SNS alarm pattern (MEDIUM confidence)
- [S3 + CloudFront vs Amplify Hosting comparison — DEV Community](https://dev.to/aws-builders/recheck-the-differences-between-aws-amplify-hosting-and-amazon-s3-amazon-cloudfront-2d2c) — Web hosting trade-offs (MEDIUM confidence)
- [Cognito vs IAM Identity Center — DEV Community](https://dev.to/mechcloud_academy/cognito-vs-iam-identity-center-which-aws-identity-service-should-you-use-4a4l) — Auth service selection (MEDIUM confidence)
- [DynamoDB Security Best Practices — AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/encryption-best-practices/dynamodb.html) — VPC endpoint, encryption (HIGH confidence)
- [AWS IoT Core Pricing](https://aws.amazon.com/iot-core/pricing/) — Message pricing, Basic Ingest cost savings (HIGH confidence)
- [API Gateway HTTP API vs REST API — AWS](https://aws.amazon.com/api-gateway/features/) — Pricing and feature comparison (HIGH confidence)

---

*Stack research for: AWS IoT Device Monitoring Platform*
*Researched: 2026-03-27*
