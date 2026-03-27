# Pitfalls Research

**Domain:** AWS IoT device monitoring platform architecture design
**Researched:** 2026-03-27
**Confidence:** HIGH (grounded in AWS Well-Architected IoT Lens, official AWS docs, and production post-mortems)

---

## Critical Pitfalls

### Pitfall 1: Ignoring Device Shadow for Offline Command Delivery

**What goes wrong:**
The architecture routes commands directly to device MQTT topics without accounting for the fact that devices are offline 99% of the time (connecting only hourly). Commands sent to a topic the device isn't subscribed to are silently dropped. The design shows a "send command" API endpoint that publishes directly to `$aws/things/{id}/command` without persistence.

**Why it happens:**
Designers think of IoT Core like a standard messaging bus. They model the happy path (device connected, command delivered) and forget the disconnected case. It's also easy to conflate device-to-cloud (telemetry) and cloud-to-device (commands) data flows, which have fundamentally different reliability requirements.

**How to avoid:**
Use AWS IoT Device Shadow as the persistence layer for all desired state / pending commands. The Shadow stores the `desired` state in the cloud. When a device connects hourly, it fetches its shadow delta (`$aws/things/{id}/shadow/update/delta`) and executes pending commands, then reports back with `reported` state. This guarantees delivery without polling, retry logic, or custom queuing.

**Warning signs in a design:**
- Diagram shows commands going directly to MQTT topics without a shadow or queue
- No mention of what happens when a device is offline at command-issue time
- "Command delivery" section omits reconnection/sync flow
- Architecture shows SQS as the command persistence layer (SQS is for fan-out/buffering, not device state synchronization)

**Documentation section to address:**
Command/configuration push section — must explicitly explain Device Shadow desired/reported state reconciliation and show the sequence: API writes `desired` → Shadow stores → device connects → delta received → command executed → device writes `reported`.

---

### Pitfall 2: Lambda Per-Message Invocation for High-Throughput Telemetry

**What goes wrong:**
The IoT Rule Engine is configured to invoke a Lambda function for every single MQTT message. At 1,000 devices sending hourly, that's 1,000 Lambda invocations per hour for batch loads, but event-driven alarms can spike. More critically, if devices send sub-hourly heartbeats or the fleet grows to tens of thousands, Lambda invocation costs and cold-start latency become prohibitive.

**Why it happens:**
IoT Rule → Lambda is the simplest path and works perfectly in demos and small-scale tests. The cost problem is invisible at low scale and the architecture looks "clean" because it's serverless.

**How to avoid:**
Route high-volume telemetry through Kinesis Data Firehose (for Data Lake delivery) or Kinesis Data Streams (for real-time processing with batched Lambda). Firehose costs per GB ingested and automatically batches records into Parquet on S3 — it eliminates the small-file problem and costs 60%+ less than per-invocation Lambda at scale. Reserve direct Lambda invocations for low-frequency, time-sensitive events (alarms, threshold breaches) where immediate processing is justified.

**Warning signs in a design:**
- IoT Rule → Lambda for all message types with no batching strategy
- No mention of Kinesis Firehose or Data Streams for telemetry path
- Lambda described as the primary processing layer for all ingested data
- No distinction between high-volume telemetry and low-volume alarm events in the processing pipeline

**Documentation section to address:**
Data processing pipeline section and cost optimization analysis — should explicitly justify Firehose vs Lambda per message type with cost rationale.

---

### Pitfall 3: Flat S3 Data Lake Without Partitioning (The $500 Athena Query)

**What goes wrong:**
IoT data lands in S3 as unpartitioned JSON files or with shallow partitioning (e.g., by day only). A single Athena query scanning 3 years of historical data to find one week of device telemetry costs $235+ because it scans the entire dataset. The architecture shows S3 as the Data Lake target without specifying a partitioning scheme or file format.

**Why it happens:**
The immediate concern is getting data into S3. Partitioning feels like an optimization to add later. Designers also default to JSON because it's the native device format, not realizing that 100 GB of JSON becomes 12 GB as Parquet with Snappy compression, and Athena query cost drops from $5.00 to $0.60.

**How to avoid:**
Design S3 prefixes as: `s3://bucket/telemetry/year=YYYY/month=MM/day=DD/device_type=X/` using Hive-style partition projection. Use Kinesis Data Firehose dynamic partitioning to write Parquet (not JSON) directly. Register partitions in AWS Glue Data Catalog. Add S3 lifecycle policies: Standard → Intelligent-Tiering → Glacier for cold data. This is the "AI-ready format" the project requires.

**Warning signs in a design:**
- Data Lake section shows raw JSON storage in S3 without conversion
- No partitioning strategy mentioned (or only temporal partitioning without device dimensions)
- Glue Crawler described as the sole schema mechanism (crawlers are slow and expensive — prefer Glue Catalog with known schemas)
- No mention of file compaction to address the small-file problem
- S3 lifecycle rules absent from cost optimization discussion

**Documentation section to address:**
ETL to Data Lake section — must show Parquet format, partition scheme, Glue Catalog registration, and Athena query cost implications.

---

### Pitfall 4: Wildcard IoT Policies (The "Star" Policy Anti-Pattern)

**What goes wrong:**
The architecture grants devices an IoT policy with `iot:*` on resource `arn:aws:iot:*:*:*`. Any compromised device certificate can then publish to any topic, subscribe to any topic, and read any device shadow in the account — including other devices' telemetry and commands.

**Why it happens:**
Overly permissive policies get the prototype working instantly. Security tightening feels like "later work." In assessments, designers often describe IAM/security without specifying policy scope, leaving the evaluator to assume it's locked down.

**How to avoid:**
Each device policy must use policy variables to scope permissions to its own resources only:
- `iot:Connect` → `clientId: ${iot:ClientId}`
- `iot:Publish` → `topic: devices/${iot:ThingName}/telemetry`
- `iot:Subscribe` → `topic: $aws/things/${iot:ThingName}/shadow/update/delta`
- `iot:Receive` → same shadow delta topic

Evaluators specifically look for policy variables (`${iot:ThingName}`) — their absence in any policy example is a red flag.

**Warning signs in a design:**
- IoT policies shown without policy variables
- Single shared policy for all devices with broad resource ARNs
- Security section describes TLS and X.509 certs but not IoT policy scope
- No mention of AWS IoT Device Defender for ongoing audit

**Documentation section to address:**
Security components section — must include a policy example showing `${iot:ThingName}` scoping. Evaluators at AWS-level roles will check this specifically.

---

### Pitfall 5: Shared Fleet Certificate (Single Claim Certificate for All Devices)

**What goes wrong:**
The provisioning design uses a single X.509 claim certificate embedded in all device firmware. If that certificate's private key is compromised, an attacker can provision unlimited rogue devices and the only recovery is revoking the certificate — breaking every device in the fleet.

**Why it happens:**
Fleet provisioning by claim is documented as using a shared bootstrap certificate, and designers stop there without implementing the exchange to per-device certificates.

**How to avoid:**
Use AWS IoT Fleet Provisioning with claim certificates that are limited to the provisioning action only (`iot:Publish` on the provisioning topic, `iot:Receive` on the provisioning response). During first connection, the device exchanges the claim certificate for a unique per-device X.509 certificate signed by AWS CA. After exchange, the claim certificate grants no production IoT permissions. Batch claim certificates per manufacturing run, not per fleet.

**Warning signs in a design:**
- Single certificate used for device authentication with no mention of per-device issuance
- Provisioning section absent entirely (common in assessments that focus on the cloud side)
- No AWS IoT Fleet Provisioning or Just-in-Time Registration (JITR) mentioned

**Documentation section to address:**
Security section and device provisioning — explain claim → per-device certificate exchange flow.

---

### Pitfall 6: Database Publicly Accessible Despite Stated Requirement

**What goes wrong:**
The design describes "private databases" but the architecture diagram shows RDS or DynamoDB without explicit VPC placement, security groups, or subnet configuration. The REST API Lambda functions are not configured with VPC access, so they reach the database via the public internet endpoint — defeating the stated security requirement.

**Why it happens:**
DynamoDB is often treated as a managed service that "doesn't need VPC" because it doesn't live in a VPC by default. Lambda functions default to no VPC attachment. Designers describe the intent (private) without designing the mechanism (VPC endpoints, subnet placement).

**How to avoid:**
- RDS/Aurora: Place in private subnets with no public IP. Attach a security group allowing inbound only from Lambda security group and application subnets.
- DynamoDB: Create a VPC Gateway Endpoint for DynamoDB so Lambda traffic never leaves the AWS network. This satisfies "internal AWS services only" without requiring NAT Gateway for DynamoDB calls.
- Lambda: Attach to VPC private subnets. Use Interface Endpoints for other AWS service calls (SSM, Secrets Manager, etc.).
- Never attach databases to public subnets, even with restrictive security groups.

**Warning signs in a design:**
- Architecture says "private database" but diagram shows no VPC subnet labels
- Lambda functions show database access with no VPC attachment noted
- DynamoDB used without mentioning VPC Gateway Endpoint
- Security section describes encryption-at-rest/in-transit but not network isolation

**Documentation section to address:**
Security components (VPC architecture), database design, and the main architecture diagram — subnet labels (public/private/data tier) must be explicit.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| IoT Rule → Lambda for all messages | Simple, no extra services | Lambda cost explosion at scale, cold-start latency on alarms | Never for high-volume telemetry; acceptable for alarm events only |
| JSON in S3 without Parquet conversion | No ETL step needed | 8x storage cost, 10x Athena query cost, not AI-ready | Never for a production Data Lake — this is a documented requirement |
| Single IoT policy for all devices | One policy to manage | Any compromise grants access to full fleet's topics | Never — policy variables are a 5-minute addition |
| Device Shadow without TTL strategy | No extra logic | Shadow state accumulates stale commands; shadow size limit (8 KB) hit silently | Acceptable if commands are designed as idempotent state deltas, not imperative commands |
| CloudWatch Alarms → SNS direct | Fewest components | Cannot do content-based routing or multi-channel enrichment without Lambda | Acceptable for MVP; EventBridge preferred for multi-channel alarm routing |
| Glue Crawler for schema discovery | Zero schema definition work | Crawlers run on schedule, miss partitions, cost per DPU-hour | Never in production — define schema explicitly in Glue Catalog |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| IoT Core → Kinesis Firehose | Putting raw JSON records without schema — Firehose cannot convert to Parquet without a Glue schema | Register schema in Glue Data Catalog before configuring Firehose record format conversion |
| IoT Core → Lambda (Rules Engine) | Not configuring an error action — failed rule executions silently drop messages | Always configure `errorAction` in IoT Rule (send to SQS DLQ or SNS) |
| Lambda → RDS in VPC | Lambda cold starts inside VPC take 10-15 seconds without provisioned concurrency or RDS Proxy | Use RDS Proxy for connection pooling; for DynamoDB prefer it over RDS for serverless workloads |
| Device Shadow → Device | Sending each state field as individual shadow updates (e.g., temperature, humidity as separate updates) | Batch all field changes into one shadow update — each update is a metered operation |
| API Gateway → Lambda → VPC | API Gateway public endpoint exposes REST API — must configure Cognito/IAM authorizer | Never deploy API Gateway without an authorizer for non-public operations |
| IoT Core → SNS for alarms | Publishing alarm topic to SNS directly without threshold evaluation — every reading triggers notification | Add IoT Rule condition `WHERE value > threshold` to filter before SNS publish |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Small files in S3 (IoT Firehose default buffer) | Athena queries slow; Glue job takes 10x expected time; S3 GET costs spike | Set Firehose buffer to 128 MB / 900 seconds; use Firehose dynamic partitioning | ~10,000+ messages/day when files average < 1 MB |
| Lambda cold starts for alarm processing | Alarm notifications delayed 10-30 seconds after threshold breach | Use provisioned concurrency for alarm Lambda; or route alarms through SNS directly from IoT Rule | Any latency-sensitive alarm path |
| DynamoDB hot partition on device ID | Throttling on popular devices; uneven capacity distribution | Use device_id + timestamp composite sort key; enable on-demand capacity mode | When 20% of devices generate 80% of traffic (common in industrial IoT) |
| Synchronous command delivery check | REST API polls for command delivery status, hammering DynamoDB | Use Device Shadow `reported` state as the delivery confirmation; expose via API | Any command volume > 100/minute |
| IoT Rule SQL full-topic scan | Rule evaluates against all topics in account — expensive at scale | Scope rules to specific topic prefixes: `FROM 'devices/+/telemetry'` not `FROM '#'` | Fleets > 1,000 devices with varied topic structures |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Hardcoded device credentials in firmware | Full fleet compromise from single device reverse-engineering | Use AWS IoT Fleet Provisioning — devices receive per-device certs at first boot, never hardcode |
| IoT policy without `${iot:ThingName}` scoping | Compromised device reads all other devices' shadows and publishes spoofed telemetry | Scope every IoT policy action to `${iot:ThingName}` and `${iot:ClientId}` variables |
| Databases in public subnets | DB accessible from internet even with security groups; increases blast radius | Place all data tier resources in private subnets with no route to internet gateway |
| Secrets in Lambda environment variables | Secrets visible in plain text in AWS Console, CloudTrail, logs | Store in AWS Secrets Manager; retrieve at runtime via SDK call from within VPC |
| No certificate rotation plan | Expired certificates cause fleet-wide outage; long-lived certs increase exposure window | Document rotation policy; enable AWS IoT Device Defender certificate expiry checks |
| CloudWatch Logs without log retention | Log groups accumulate indefinitely; cost grows unbounded | Set retention on all log groups (30 days for operational, 90 days for security audit) |

---

## Documentation Quality Red Flags

Things an experienced AWS architect would flag when reviewing this type of assessment:

| Red Flag | What It Signals | What Good Looks Like |
|----------|-----------------|----------------------|
| Diagram shows components but no data flow direction | Designer understands services but not integration patterns | Arrows with labels: message format, protocol, trigger type |
| "Use DynamoDB for IoT data" without justification | Cargo-culting AWS services without understanding tradeoffs | Comparison table: DynamoDB vs Timestream vs RDS for IoT telemetry — with access pattern analysis |
| Security section lists services (KMS, IAM, VPC) without explaining configuration | Naming services is not architecture | Explain what each service controls, show policy snippets or group rules |
| Cost section says "use serverless to save costs" | Misunderstanding of serverless pricing | Show specific cost drivers (Firehose per GB vs Lambda per invocation at N devices) |
| Disconnected device handling is one sentence | This is the hardest part of the design — glossing over it shows shallow analysis | Full sequence: device connects → fetches shadow delta → executes command → reports state → API reflects delivery |
| No Mermaid diagram for the command delivery flow | This specific flow has many states (offline, connecting, executing, confirmed) | Sequence diagram showing all actors: Web UI → API → Shadow → Device → Shadow → API |
| Single architecture diagram with no drill-down | High-level is fine but omits implementation detail | One overview diagram + per-subsystem diagrams (ingestion, processing, command delivery, Data Lake) |

---

## "Looks Done But Isn't" Checklist

- [ ] **Device Shadow**: Shows desired/reported reconciliation — not just "device reads shadow on connect". Verify the delta subscription topic is explicitly shown.
- [ ] **VPC design**: Subnet tiers labeled public/private/data in diagram. Verify Lambda VPC attachment is documented for DB-accessing functions.
- [ ] **Data Lake**: Shows Parquet output with partition scheme, not just "S3 bucket". Verify Glue Data Catalog is part of the ETL description.
- [ ] **Alarm routing**: Shows threshold evaluation in IoT Rule SQL, not just "IoT publishes to SNS". Verify alarm deduplication is addressed (same alarm firing every minute).
- [ ] **IoT policies**: Includes policy variable example (`${iot:ThingName}`). Verify the policy is not `iot:*` on `*`.
- [ ] **REST API security**: Shows authentication mechanism (Cognito User Pools or IAM SigV4). Verify API Gateway authorizer is configured.
- [ ] **Cost optimization**: Names specific service cost drivers. Verify it doesn't just say "use serverless" without quantification.
- [ ] **Error handling**: Shows what happens when IoT Rule fails. Verify `errorAction` is mentioned.

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Commands dropped to offline devices | HIGH | Audit missed commands from DynamoDB command log; re-issue via Shadow desired state; requires command idempotency design from day 1 |
| Unpartitioned S3 Data Lake at scale | MEDIUM | Rewrite S3 objects with Glue ETL job into partitioned Parquet; update Glue Catalog; one-time cost but weeks of effort |
| Wildcard IoT policy exploited | HIGH | Revoke all device certificates; re-provision fleet with scoped policies; near-impossible at scale without fleet provisioning infrastructure |
| Lambda per-message cost explosion | MEDIUM | Replace IoT Rule Lambda action with Kinesis Firehose action; migrate Lambda processing logic to Kinesis consumer; 1-2 sprints |
| Public database endpoint exposed | HIGH | Migrate to private subnet, update security groups, configure VPC endpoints; requires application downtime |
| Single claim certificate compromised | HIGH | Revoke claim cert (blocks all future provisioning); new cert requires firmware update to entire fleet |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| No Device Shadow for offline commands | Architecture design phase | Command delivery sequence diagram shows Shadow desired/delta/reported flow |
| Lambda per-message (no batching) | Architecture design phase (processing pipeline) | Processing pipeline diagram distinguishes telemetry path (Firehose) from alarm path (Lambda) |
| Unpartitioned/non-Parquet Data Lake | Data Lake / ETL design phase | ETL section specifies Parquet format, S3 prefix scheme with partition keys |
| Wildcard IoT policies | Security design phase | Security section includes IoT policy example with `${iot:ThingName}` |
| Shared fleet certificate (no per-device) | Security design phase | Provisioning section describes Fleet Provisioning claim-to-per-device exchange |
| Database publicly accessible | Network/VPC design phase | Architecture diagram labels private subnets; VPC Gateway Endpoint for DynamoDB noted |
| Missing error actions on IoT Rules | Processing pipeline design phase | IoT Rule definitions include `errorAction` pointing to DLQ |
| Alarm flooding (no threshold SQL) | Alarm notification design phase | IoT Rule SQL shown with `WHERE` clause for threshold evaluation |

---

## Sources

- [AWS IoT Well-Architected Lens — IoT Lens](https://docs.aws.amazon.com/wellarchitected/latest/iot-lens/iot-lens.html)
- [AWS IoT Device Shadow Service — Well-Architected Lens](https://docs.aws.amazon.com/wellarchitected/latest/iot-lens/aws-iot-device-shadow-service.html)
- [Security best practices in AWS IoT Core](https://docs.aws.amazon.com/iot/latest/developerguide/security-best-practices.html)
- [Implementing time-critical cloud-to-device IoT message patterns on AWS IoT Core](https://aws.amazon.com/blogs/iot/implementing-time-critical-cloud-to-device-iot-message-patterns-on-aws-iot-core/)
- [Building a Real-Time Data Lake on AWS: S3, Glue, and Athena in Production](https://dev.to/aws-builders/building-a-real-time-data-lake-on-aws-s3-glue-and-athena-in-production-4g0b)
- [Cost Optimization for Large Datasets on AWS: A Case for S3, Athena, and Glue](https://stratus10.com/blog/managing-large-datasets-on-aws)
- [Design Practices: AWS IoT Solutions](https://www.iotforall.com/aws-iot-solutions-design-practice)
- [Designing Scalable IoT Solutions on AWS](https://www.iotforall.com/aws-iot-solutions-design)
- [How to automate onboarding of IoT devices to AWS IoT Core at scale with Fleet Provisioning](https://aws.amazon.com/blogs/iot/how-to-automate-onboarding-of-iot-devices-to-aws-iot-core-at-scale-with-fleet-provisioning/)
- [How to ensure resilience for your AWS IoT Rules Engine to AWS Lambda integration](https://dev.to/iotbuilders/how-to-ensure-resilience-for-your-aws-iot-rules-engine-to-aws-lambda-integration-1aoi)
- [Architectural Guidance for High-Throughput IoT with SQS and Kinesis](https://repost.aws/questions/QU9M_0dzNrQfuLEaQzPIRO2w/architectural-guidance-and-optimization-insights-for-high-throughput-iot-workload-on-aws-with-sqs-and-kinesis)
- [Serverless Cost Optimization: Kinesis Streams vs Firehose](https://lumigo.io/blog/serverless-cost-optimization-kinesis-streams-vs-firehose/)
- [Common architecture patterns to securely connect IoT devices to AWS using private networks](https://aws.amazon.com/blogs/iot/common-architecture-patterns-to-securely-connect-iot-devices-to-aws-using-private-networks/)
- [AWS Industrial IoT Architecture Patterns Whitepaper](https://docs.aws.amazon.com/pdfs/whitepapers/latest/industrial-iot-architecture-patterns/industrial-iot-architecture-patterns.pdf)

---
*Pitfalls research for: AWS IoT device monitoring platform architecture design*
*Researched: 2026-03-27*
