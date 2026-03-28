## Cost Analysis

This section provides monthly cost estimates for the IoT monitoring platform at two scale points (1,000 and 10,000 devices), identifies the dominant cost drivers, and documents seven cost optimization strategies. All figures use AWS public pricing as of 2025. Actual costs vary by region, usage patterns, and reserved capacity commitments.

---

### Scale Assumptions

| Parameter | Low (1,000 devices) | High (10,000 devices) |
|-----------|---------------------|----------------------|
| Devices | 1,000 | 10,000 |
| Telemetry frequency | 1 message/device/hour | 1 message/device/hour |
| Messages/month | ~720,000 | ~7,200,000 |
| Avg message size | ~1 KB | ~1 KB |
| API requests/month | ~100,000 | ~1,000,000 |
| Concurrent dashboard users | 5–10 | 20–50 |

---

### Monthly Cost Estimate

| Service | 1,000 devices/month | 10,000 devices/month | Notes |
|---------|---------------------|----------------------|-------|
| AWS IoT Core | $1–$3 | $10–$30 | With Basic Ingest (~50% savings vs standard) |
| Kinesis Data Streams | $5–$10 | $20–$50 | On-Demand mode |
| Kinesis Data Firehose | $1–$3 | $10–$25 | Includes Parquet conversion |
| Lambda (all functions) | $1–$5 | $5–$20 | Graviton2 arm64, batch processing |
| Amazon Timestream | $1–$5 | $5–$25 | 24h memory + 1yr magnetic retention |
| Amazon DynamoDB | $1–$5 | $5–$20 | On-Demand, TTL for transient data |
| Aurora Serverless v2 | $10–$20 | $20–$50 | 0.5 ACU minimum (~$43/month at 0.5 ACU 24/7); scales with API load |
| S3 + Glue + Athena | $3–$10 | $10–$30 | Parquet partitioned, pay-per-query |
| API Gateway HTTP API | $1–$3 | $3–$10 | $1.00/million requests |
| CloudFront | $1–$3 | $3–$8 | SPA assets + API responses, gzip/Brotli |
| Supporting services* | $15–$25 | $20–$35 | See breakdown below |
| **TOTAL** | **$40–$92** | **$111–$303** | |

**Supporting services breakdown:**

- VPC Interface Endpoints: ~3 endpoints (Timestream, Kinesis, Secrets Manager) × $7/month × 2 AZs = ~$42/month
- AWS WAF: $5/WebACL/month + $1/million requests
- AWS KMS: $1/CMK/month × 3–5 keys = $3–$5/month
- AWS Secrets Manager: $0.40/secret/month × 5 secrets = ~$2/month
- AWS CloudTrail: Free for management events
- Amazon CloudWatch: Free tier covers basic monitoring; $0.30/metric/month above free tier

> **Important:** VPC Interface Endpoints are the dominant fixed cost at low device counts ($42/month). This cost is independent of message volume. Gateway Endpoints (S3, DynamoDB) are free. At production scale, consider whether all Interface Endpoints are justified vs alternative access patterns. See [01-security-foundation.md](01-security-foundation.md) for the VPC endpoint configuration.

---

### Environment Costs

Dev/staging environments cost approximately 10–20% of production:

- Aurora Serverless v2 scales to minimum 0.5 ACU when idle (~$43/month at minimum)
- Kinesis Data Streams On-Demand has no minimum charge — $0 at zero traffic
- Lambda charges $0 at zero traffic
- IoT Core charges only when devices are connected and publishing

**Estimated dev/staging cost: $15–$30/month** for a lightly loaded environment with minimal device simulation.

---

### Cost Optimization Strategies

| # | Strategy | Estimated Savings | How It Works |
|---|----------|-------------------|--------------|
| 1 | IoT Core Basic Ingest | ~50% on messaging cost | Uses reserved `$aws/rules/` topic prefix to bypass the pub/sub broker for device-to-cloud telemetry. Saves ~$0.50/million messages. Requires device firmware to publish to the reserved topic path instead of a custom topic. |
| 2 | Graviton2 (arm64) Lambda | ~20% cheaper, 19% better performance | All Lambda functions configured with `arm64` architecture. No code changes required for Python/Node.js runtimes. Applied to all Lambda functions: telemetry-processor, alarm-evaluator, ses-sender, API handlers. |
| 3 | Parquet + Hive-style partitioning | 80–90% Athena scan cost reduction | Athena scans only the required partitions (`year=/month=/day=/device_type=`) instead of the full table. Parquet columnar format further reduces bytes scanned per query vs raw JSON. Enabled by Kinesis Firehose dynamic partitioning and Glue ETL Bronze→Silver conversion. |
| 4 | S3 Intelligent-Tiering | Automatic savings on aging data | Bronze zone data older than 30 days auto-moves to Infrequent Access tier (~40% cheaper). No retrieval fees for Intelligent-Tiering. Write-once raw records are ideal candidates — rarely read after initial processing. |
| 5 | DynamoDB On-Demand + TTL | Zero capacity planning + free deletions | On-Demand eliminates over-provisioning risk at variable IoT workload patterns. TTL automatically deletes expired items (command queue entries, deduplication records with 15-min TTL) at no cost. |
| 6 | Aurora Serverless v2 scale-to-zero | ~$43/month minimum vs ~$60/month provisioned | Scales to 0.5 ACU when idle (minimum billed unit, not truly zero). Provisioned db.t3.medium costs ~$60/month 24/7. Significant savings in dev/staging environments and off-hours production lulls. |
| 7 | CloudFront compression + caching | Reduce origin requests 80%+ | gzip/Brotli compression on all JS/CSS/HTML SPA assets (configured via CloudFront response headers policy). Immutable static assets (hashed filenames) cached with `Cache-Control: max-age=31536000, immutable`. Reduces S3 GET requests and CloudFront data transfer cost. |

---

### Cross-References

See each per-layer document (01–09) for detailed service configuration. See [10-overview-and-sequences.md](10-overview-and-sequences.md) for the complete system diagram and end-to-end flow sequences.
