# Phase 4: API, Web Frontend & Documentation Quality - Research

**Researched:** 2026-03-28
**Domain:** AWS API Gateway, Cognito, CloudFront/S3 OAC, Mermaid documentation, cost analysis
**Confidence:** HIGH

---

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**API Layer**
- D-01: API Gateway HTTP API (v2) + Lambda handlers in VPC private subnet. Lambda accesses Timestream, DynamoDB, Aurora via VPC endpoints. RDS Proxy mediates Lambda → Aurora connections.
- D-02: Endpoints grouped by resource domain (devices, telemetry, alarms, users/auth). Representative endpoints per group — not an exhaustive spec.
- D-03: Cognito User Pools for user authentication with JWT-based authorization. Sequence diagram required: browser → Cognito Hosted UI → JWT → API Gateway JWT authorizer → Lambda. Cognito Identity Pools documented as optional (direct frontend → AWS service access), not primary pattern.
- D-04: Comparison table: API Gateway HTTP API v2 vs REST API v1 vs App Runner. Axes: cost, JWT auth, VPC Link, usage plans, latency, cold start.
- D-05: API Gateway within AWS platform. Custom domain + ACM certificate pattern documented.

**Web Frontend**
- D-06: Comparison table: SPA S3+CloudFront vs Amazon Managed Grafana vs AWS Amplify Hosting. SPA recommended; Grafana noted for ops-only teams; Amplify as S3+CF wrapper.
- D-07: CloudFront Origin Access Control (OAC) with S3 bucket policy snippet. ACM certificate. WAF on CloudFront (inheriting Phase 1 SEC-06).
- D-08: Web-to-device command flow sequence diagram: UI button → API Gateway → Lambda → IoT Device Shadow `desired` state → cross-reference Phase 1 delta sequence.

**Overview Diagram**
- D-09: One high-level overview Mermaid diagram showing all architecture layers (IoT Ingestion → Processing Pipeline → Storage → Data Lake → API → Web Frontend → Notifications) with data flow arrows. VPC boundary and security elements (KMS, WAF) as overlay annotations.
- D-10: All existing per-layer diagrams (docs 01–07) retained unchanged. Overview is additive.
- D-11: Per-layer diagrams for API and Web Frontend layers added as new content.

**Sequence Diagrams**
- D-12: Three key-flow sequence diagrams:
  1. Telemetry ingestion end-to-end: Device → IoT Core → Rules Engine → Kinesis → Lambda → Timestream + DynamoDB
  2. Alarm notification pipeline: Device → IoT Core → Lambda evaluator → DynamoDB dedup → SNS → SES email
  3. Web-to-device command delivery: Browser → Cognito → API Gateway → Lambda → IoT Device Shadow → device reconnect → delta apply
- D-13: `sequenceDiagram` Mermaid syntax. Each diagram cross-references the detailed section where each component is documented.

**Technology Comparison Tables**
- D-14: Audit all phases — tables required for: IoT entry point (Ph1), stream buffer (Ph2), time-series store (Ph2), relational store (Ph2), ETL trigger (Ph3), query engine (Ph3), API front door (Ph4 NEW), web hosting (Ph4 NEW), auth provider (Ph4 NEW).
- D-15: Consistent format: Alternative | Pros | Cons | Recommendation. Prior docs tables referenced, not duplicated.

**Cost Analysis**
- D-16: Per-service-category cost table, monthly range for 1,000–10,000 devices at hourly telemetry. All categories: IoT Core, Kinesis, Lambda, Timestream, DynamoDB, Aurora Serverless v2, S3+Glue+Athena, API Gateway, CloudFront, supporting services.
- D-17: Single low–high production range. Dev/staging ~10–20% of production note. Align with $40–$145/month established in project research.
- D-18: Cost optimization subsection: top 5–7 strategies with estimated savings.

**Documentation Format**
- D-19: Continue pattern: narrative → Mermaid diagram → comparison table → design notes.
- D-20: New files: 08-api-layer.md, 09-web-frontend.md, 10-overview-and-sequences.md, 11-cost-analysis.md. Table of contents / index file.

### Claude's Discretion

- Exact Mermaid styling, color coding, and node arrangement in overview diagram
- Level of detail in API endpoint listing (number of representative endpoints per resource)
- Whether to include Cognito Hosted UI configuration details or keep at architectural level
- CloudFront cache policy specifics (TTL values, behavior patterns)
- Whether comparison tables from prior phases need a summary roll-up table in the cost section
- Exact layout of the final document index

### Deferred Ideas (OUT OF SCOPE)

None from discussion. v2 bonus items remain deferred:
- MCLD-01/02/03: Multi-cloud equivalents (Azure, GCP, Kubernetes)
- ADV-01..05: Device Defender, Greengrass, CI/CD, DR/multi-region

</user_constraints>

---

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| API-01 | Architecture documents API Gateway (HTTP API) + Lambda for REST API | D-01, D-02, D-04. Pattern: HTTP API v2 with Lambda integration, VPC Link to private subnet. Endpoint grouping by resource domain. |
| API-02 | Architecture documents Cognito for user authentication and JWT-based authorization | D-03. Cognito User Pools → JWT → API Gateway JWT authorizer. Full sequence diagram required. |
| API-03 | Architecture documents API deployed within the AWS platform (not external) | D-05. API Gateway is AWS-managed; Lambda runs in VPC private subnet. Custom domain + ACM. |
| WEB-01 | Architecture documents S3 + CloudFront for SPA hosting (or Managed Grafana alternative with comparison) | D-06, D-07. S3+CloudFront with OAC. Comparison table with Grafana and Amplify alternatives. |
| WEB-02 | Architecture documents web-to-device command flow (UI → API → Device Shadow) | D-08. Full sequenceDiagram cross-referencing doc 03 Device Shadow sequence. |
| WEB-03 | Architecture documents CloudFront OAC for secure S3 origin access | D-07. OAC replaces OAI. S3 bucket policy snippet restricts access to CloudFront only. |
| DOC-01 | Architecture includes Mermaid diagram of the complete system (all layers and data flows) | D-09, D-10. One top-level `flowchart TD` or `graph LR` spanning all 8 layers. |
| DOC-02 | Architecture includes per-layer Mermaid diagrams with component detail | D-11. Two new per-layer diagrams: API layer and Web Frontend layer. Docs 01–07 already have per-layer diagrams. |
| DOC-03 | Architecture includes technology comparison tables for each major decision (alternatives, pros, cons, recommendation) | D-14, D-15. Audit all prior docs; add three new tables in Phase 4 docs (API front door, web hosting, auth provider). |
| DOC-04 | Architecture includes cost optimization analysis with estimated monthly cost range | D-16, D-17, D-18. Per-service table + optimization strategies in 11-cost-analysis.md. |
| DOC-05 | Architecture includes sequence diagrams for key flows (telemetry ingestion, alarm notification, command delivery) | D-12, D-13. Three sequenceDiagram blocks in 10-overview-and-sequences.md. |

</phase_requirements>

---

## Summary

Phase 4 completes the architecture document. It adds four new markdown files (08-api-layer.md, 09-web-frontend.md, 10-overview-and-sequences.md, 11-cost-analysis.md) and optionally an index file. All design decisions are locked in CONTEXT.md, so this phase is primarily a documentation assembly task: every component has been designed in prior phases; Phase 4 writes the API and web frontend descriptions, then synthesizes all layers into cross-cutting quality artifacts.

The technical content for Phase 4 is well-understood from the project's established stack (CLAUDE.md, STACK.md, SUMMARY.md) and from the seven existing architecture documents. The API layer inherits its VPC topology from doc 01, its Lambda patterns from doc 04, and its Cognito authentication from the locked decisions. The web frontend inherits WAF from doc 01 (SEC-06). The three sequence diagrams cross-reference existing component descriptions — they are narrative connective tissue, not new design decisions. The cost analysis draws figures already stated across prior documents and CLAUDE.md's cost optimization table.

The key planner concerns are: (1) exact document structure for the four new files, (2) the complete overview Mermaid diagram that spans all eight layers without becoming unreadable, and (3) ensuring the three sequence diagrams reference the correct doc numbers (01–07) as established.

**Primary recommendation:** Plan four tasks, one per new document, with the overview diagram and sequences file as the synthesis capstone. The index/README file should be the last task, linking all 11 documents.

---

## Standard Stack

### Core (Phase 4 — documentation assembly, no new services)

| Layer | Service | Established In | Phase 4 Role |
|-------|---------|---------------|--------------|
| API front door | API Gateway HTTP API v2 | CONTEXT.md D-01 | Document endpoint groups, VPC Link, JWT authorizer |
| API auth | Amazon Cognito User Pools | CONTEXT.md D-03 | Sequence diagram + auth comparison table |
| API compute | AWS Lambda (Python 3.12, arm64, VPC) | doc 04 | Document handler → storage path via VPC endpoints |
| API ↔ Aurora | Amazon RDS Proxy | Phase 2 decision | Document connection pooling, mandatory for Lambda→Aurora |
| SPA hosting | Amazon S3 (private bucket) | CONTEXT.md D-07 | Document OAC bucket policy, no public access |
| CDN | Amazon CloudFront (OAC) | CONTEXT.md D-07 | Document distribution config, cache behaviors, WAF attach |
| WAF | AWS WAF v2 WebACL | doc 01 SEC-06 | Cross-reference only — already documented |
| TLS | AWS Certificate Manager (ACM) | CONTEXT.md D-05 | Note custom domain + auto-renewal |
| Diagram syntax | Mermaid `flowchart TD` / `sequenceDiagram` | All prior docs | Continue established style |

### Supporting (cost analysis inputs, all previously documented)

| Service | Cost Driver | Per-service pricing source |
|---------|-------------|---------------------------|
| AWS IoT Core | $1.00/million messages (5 KB increments) | AWS IoT Core Pricing page |
| Kinesis Data Streams | $0.015/shard-hour provisioned; On-Demand pay-per-use | AWS Kinesis Pricing |
| Kinesis Data Firehose | $0.029/GB ingested | AWS Kinesis Pricing |
| AWS Lambda | $0.0000002/req + $0.0000166667/GB-sec | AWS Lambda Pricing |
| Amazon Timestream | Writes $0.50/M; Queries $0.01/GB; Magnetic $0.03/GB-month | AWS Timestream Pricing |
| Amazon DynamoDB | On-Demand: $1.25/M write RCU, $0.25/M read RCU | AWS DynamoDB Pricing |
| Amazon S3 | $0.023/GB-month Standard | AWS S3 Pricing |
| AWS Glue | $0.44/DPU-hour | AWS Glue Pricing |
| Amazon Athena | $5/TB scanned | AWS Athena Pricing |
| API Gateway HTTP API | $1.00/million requests | AWS API Gateway Pricing |
| Amazon CloudFront | $0.0085/GB (first 10 TB) | AWS CloudFront Pricing |
| Aurora Serverless v2 | $0.12/ACU-hour (min 0.5 ACU) | AWS Aurora Pricing |
| Amazon SNS | $0.50/million messages | AWS SNS Pricing |
| Amazon SES | $0.10/1,000 emails | AWS SES Pricing |
| VPC Interface Endpoints | ~$7/month per AZ per endpoint | AWS PrivateLink Pricing |
| AWS KMS | $1/CMK/month + $0.03/10K API calls | AWS KMS Pricing |
| AWS Secrets Manager | $0.40/secret/month + $0.05/10K API calls | AWS Secrets Manager Pricing |
| AWS WAF | $5/WebACL/month + $1/million requests | AWS WAF Pricing |
| Amazon CloudWatch | Free tier 10 metrics, 3 dashboards; $0.30/metric/month above | AWS CloudWatch Pricing |

---

## Architecture Patterns

### New Document Structure (Phase 4 additions)

```
docs/
├── architecture/
│   ├── 01-security-foundation.md        [existing — do not modify]
│   ├── 02-device-connectivity-ingestion.md  [existing]
│   ├── 03-device-management.md          [existing — has Device Shadow sequence]
│   ├── 04-data-pipeline-processing.md   [existing]
│   ├── 05-storage-layer.md              [existing]
│   ├── 06-alarm-notifications.md        [existing]
│   ├── 07-data-lake-etl.md              [existing]
│   ├── 08-api-layer.md                  [NEW — Phase 4]
│   ├── 09-web-frontend.md               [NEW — Phase 4]
│   ├── 10-overview-and-sequences.md     [NEW — Phase 4]
│   ├── 11-cost-analysis.md              [NEW — Phase 4]
│   └── README.md                        [NEW — index with links to all 11 docs]
```

### Pattern 1: API Layer Document (08-api-layer.md)

**What:** Narrative description of the REST API: endpoint groups, VPC Lambda handler pattern, Cognito JWT flow, RDS Proxy for Aurora connections, VPC Link for private target routing, and three comparison tables (API front door, auth provider).

**Structure:**
1. Section intro — API architecture role in the platform
2. API Gateway HTTP API v2 configuration notes (JWT authorizer, custom domain, ACM)
3. Endpoint group table (devices, telemetry, alarms, users/auth) with 1–2 representative endpoints per group and their backing storage
4. Per-layer Mermaid diagram: CloudFront → API Gateway → Lambda (VPC) → Timestream/DynamoDB/Aurora (Data tier) via VPC endpoints
5. Cognito authentication sequence diagram (D-03)
6. VPC Link pattern note (API Gateway → NLB/ALB → Lambda in private subnet)
7. Comparison table: API Gateway HTTP API v2 vs REST API v1 vs App Runner (D-04)
8. Comparison table: Auth provider — Cognito User Pools vs IAM Identity Center vs self-managed IdP
9. Design notes: cold starts, RDS Proxy mandatory, Lambda timeout limits (15 min), arm64 runtime

**Cognito JWT Flow (for sequenceDiagram in doc 08 or doc 10):**
```
Browser → Cognito Hosted UI: authenticate (username+password or social)
Cognito Hosted UI → Browser: authorization code (PKCE)
Browser → Cognito Token Endpoint: exchange code for tokens
Cognito Token Endpoint → Browser: id_token + access_token (JWT, signed RS256)
Browser → API Gateway HTTP API: request + Authorization: Bearer {access_token}
API Gateway JWT Authorizer → Cognito JWKS endpoint: validate signature + claims
API Gateway → Lambda: invoke (with event.requestContext.authorizer.claims)
Lambda → Storage (Timestream / DynamoDB / Aurora via RDS Proxy): query
Lambda → API Gateway: response
API Gateway → Browser: HTTP 200 JSON
```

### Pattern 2: Web Frontend Document (09-web-frontend.md)

**What:** SPA hosting architecture — S3 private bucket, CloudFront distribution with OAC, WAF WebACL attachment, ACM certificate, custom domain, and the web-to-device command flow sequence diagram.

**Structure:**
1. Section intro — SPA role, why custom UI over Grafana
2. Per-layer Mermaid diagram: User → CloudFront (WAF) → S3 (OAC) for static assets; User → API Gateway for dynamic data
3. CloudFront OAC configuration notes — replaces OAI; bucket policy snippet
4. Cache behavior table: `/` → S3 (HTML, short TTL); `/static/*` → S3 (long TTL); `/api/*` → not cached (forwarded to API Gateway)
5. Web-to-device command flow sequenceDiagram (D-08)
6. Comparison table: SPA S3+CloudFront vs Amazon Managed Grafana vs AWS Amplify Hosting (D-06)
7. Design notes: WAF cross-reference (doc 01), ACM wildcard cert, Route 53 alias record

**OAC Bucket Policy Snippet pattern (to include verbatim in doc):**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipal",
    "Effect": "Allow",
    "Principal": {
      "Service": "cloudfront.amazonaws.com"
    },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-spa-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DISTRIBUTION_ID"
      }
    }
  }]
}
```
**Confidence:** HIGH — from AWS prescriptive guidance on OAC.

### Pattern 3: Overview and Sequences Document (10-overview-and-sequences.md)

**What:** The synthesis document. Contains the single top-level architecture diagram spanning all eight layers, plus the three sequence diagrams for telemetry ingestion, alarm notification, and web-to-device command flow.

**Top-level overview Mermaid approach:**
- Use `flowchart TD` (top-down layout) or `flowchart LR` (left-right) — `LR` is more readable for a multi-layer horizontal pipeline.
- Group services into `subgraph` blocks per architectural layer.
- Show only major data flows — not every API call. Seven arrows suffice: Device → IoT Core → Kinesis → Lambda → Storage; Lambda → SNS; Browser → CloudFront → API GW → Lambda.
- VPC boundary as one large outer `subgraph VPC[...]` with private and data subnet subgraphs nested inside.
- Security overlays (KMS, WAF) as annotation nodes connected to affected services with dashed arrows or `style` class coloring rather than separate boxes.

**Layer ordering for overview diagram (top-to-bottom or left-to-right):**
```
Layer 1: IoT Devices (external, outside VPC)
Layer 2: IoT Ingestion (AWS IoT Core, Rules Engine)
Layer 3: Message Processing (Kinesis Data Streams, Kinesis Firehose, Lambda)
Layer 4: Storage — Hot (Timestream, DynamoDB, Aurora via RDS Proxy)
Layer 5: Data Lake — Cold (S3 Bronze/Silver/Gold, Glue ETL, Athena)
Layer 6: Notifications (SNS, SES, EventBridge)
Layer 7: REST API (API Gateway HTTP API, Lambda VPC handlers)
Layer 8: Web Frontend (CloudFront/WAF, Cognito, SPA on S3)
```

**Three sequence diagrams (D-12, D-13):**

Sequence 1 — Telemetry ingestion end-to-end:
- Participants: Device, IoT Core, Rules Engine, Kinesis DS, Lambda (telemetry-processor), Timestream, DynamoDB, Kinesis Firehose, S3 Bronze
- Key steps: Device publishes to `devices/{thingName}/telemetry` → Rules Engine matches TelemetryRule → fan-out to Kinesis DS (hot path) AND Kinesis Firehose (cold path) → Lambda batch consumer writes to Timestream + DynamoDB latest-value → Firehose delivers raw JSON to S3 Bronze
- Cross-reference: docs 02, 04, 05

Sequence 2 — Alarm notification pipeline:
- Participants: Device, IoT Core, Rules Engine, Lambda (alarm-evaluator), DynamoDB (dedup), SNS, Lambda (ses-email-sender), SES, EventBridge
- Key steps: Device publishes to `devices/{thingName}/alarm` → AlarmRule WHERE clause → Lambda async invocation → per-device threshold check vs Aurora → DynamoDB conditional write (dedup) → SNS publish → SES email send → EventBridge for extensibility
- Cross-reference: docs 02, 06
- Note: include dedup conditional write step explicitly (ALRM-04 evaluator checkpoint)

Sequence 3 — Web-to-device command delivery (already partially exists in doc 03 — Phase 4 adds the browser/API Gateway/Cognito front half):
- Participants: Browser, Cognito, CloudFront, API Gateway, Lambda (command-handler), IoT Shadow, DynamoDB (audit), Device
- Key steps: Browser authenticates → JWT acquired → POST /devices/{thingName}/commands → API GW JWT authorizer validates → Lambda updates Shadow desired → 202 Accepted → device connects hourly → receives delta → applies → reports back
- Cross-reference: docs 03, 08

### Pattern 4: Cost Analysis Document (11-cost-analysis.md)

**Structure:**
1. Section intro — cost philosophy (serverless = zero idle, pay-per-use)
2. Scale assumption table: 1,000 devices (low), 10,000 devices (high); hourly telemetry; 1 message/device/hour; avg message size 1 KB
3. Per-service cost table with low/high monthly range
4. Total monthly estimate: $40–$145/month (low), up to ~$300–500/month (high-end 10K devices with sustained API load) — aligned with CLAUDE.md research
5. Dev/staging cost note: ~10–20% of production
6. Cost optimization strategies subsection (D-18)

**Cost optimization strategies (top 7 with savings estimates):**
1. IoT Core Basic Ingest (~50% messaging cost) — skips broker pub/sub for device-to-cloud telemetry
2. Graviton2 (arm64) Lambda runtime (~20% cheaper, 19% better perf vs x86_64)
3. Parquet + Hive-style partitioning for Athena (80–90% scan cost reduction vs full-table JSON scan)
4. S3 Intelligent-Tiering for Bronze zone (automatic to cheaper tiers after 30 days)
5. DynamoDB On-Demand + TTL for transient data (command queue, dedup records — TTL deletes are free)
6. Aurora Serverless v2 scale-to-zero (~0.5 ACU minimum when idle vs always-on provisioned RDS)
7. CloudFront compression (gzip/Brotli on JS/CSS assets) + cache-hit maximization (reduce origin requests)

### Anti-Patterns to Avoid

- **Duplicating comparison tables from prior docs:** D-15 explicitly states prior tables are referenced, not duplicated. The overview sequences doc should link to the relevant per-layer doc for each table.
- **Making the overview diagram too granular:** Including every Lambda function or DynamoDB table in the top-level diagram makes it unreadable. Stick to service-level nodes (max 20–25 nodes in the overview). Detailed components live in per-layer diagrams.
- **Using OAI instead of OAC for CloudFront:** OAI (Origin Access Identity) is the legacy pattern. OAC (Origin Access Control) is the current AWS recommendation and required by D-07. The bucket policy syntax differs between the two — use the OAC service principal `cloudfront.amazonaws.com` with `AWS:SourceArn` condition.
- **Omitting RDS Proxy in the API Lambda → Aurora connection:** This is a Phase 2 decision that must appear in the API layer doc. Lambda → Aurora without RDS Proxy causes connection pool exhaustion at moderate concurrency (Lambda scales to hundreds of concurrent executions, each opening a new DB connection; Aurora max_connections is ~90 for 0.5 ACU).
- **Conflating Cognito User Pools and Cognito Identity Pools:** User Pools issue JWTs for application authentication. Identity Pools grant temporary IAM credentials for direct AWS service access. D-03 locks User Pools as the primary pattern; Identity Pools are documented as optional.
- **Leaving cost analysis at vague "low cost" language:** Evaluators expect specific dollar ranges per service. Use the concrete figures from CLAUDE.md and the pricing sources already researched.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JWT token validation | Custom Lambda authorizer with JWKS fetch | API Gateway HTTP API native JWT authorizer | HTTP API has built-in JWT validation pointing at Cognito User Pool. No Lambda needed, no cold starts on auth path, no token caching bugs. |
| S3 origin access restriction | IP whitelist or bucket public access | CloudFront Origin Access Control (OAC) | OAC is the AWS-managed mechanism. Public S3 buckets fail the security requirement. IP whitelist is brittle (CloudFront IPs change). |
| DB connection pool for Lambda | Per-invocation connection open/close | Amazon RDS Proxy | Lambda→Aurora without proxy causes connection storm at any meaningful concurrency level. RDS Proxy is not optional for this pattern. |
| Custom MQTT broker | EMQX or Mosquitto on EC2 | AWS IoT Core | Already decided in Phase 1. Referenced here because the API doc should note that web commands go through the Shadow service, not a custom MQTT publisher. |
| Per-endpoint auth logic | Custom middleware in each Lambda handler | API Gateway JWT authorizer at the integration level | Auth enforced at the gateway means a misconfigured Lambda cannot accidentally serve unauthenticated requests. |

---

## Common Pitfalls

### Pitfall 1: Overview Mermaid Diagram Becomes Unreadable
**What goes wrong:** Adding every service, Lambda function, DynamoDB table, and VPC endpoint to the top-level diagram creates a 50+ node graph that renders as a tangled web.
**Why it happens:** Desire for completeness conflicts with the 30,000-foot overview purpose.
**How to avoid:** Cap the overview at service-type level (e.g., "Lambda Functions" as one node, not 5 separate Lambda nodes). Show data flow direction only — no configuration details. Reserve detail for per-layer diagrams.
**Warning signs:** If the Mermaid source exceeds ~60 lines, the diagram is too detailed for an overview.

### Pitfall 2: Sequence Diagrams Do Not Cross-Reference Correctly
**What goes wrong:** Sequence diagrams reference components by informal names that don't match the component names in the per-layer docs, making cross-references confusing for the evaluator.
**How to avoid:** Use the exact service/component names established in prior docs. For example: "Lambda (telemetry-processor)" not just "Lambda". Reference the exact doc number: "(see 04-data-pipeline-processing.md)".

### Pitfall 3: Cost Analysis Uses Only the Lowest Scale
**What goes wrong:** Stating "$40/month" without the high-end figure (10,000 devices) undersells the architecture's scalability story.
**How to avoid:** Present both the 1,000-device and 10,000-device scenarios as a range. The CLAUDE.md research already has a $40–$145/month estimate for "thousands of devices at modest load" — use this as the base and extrapolate proportionally for the high end.

### Pitfall 4: Cognito Sequence Missing PKCE / Hosted UI Steps
**What goes wrong:** Showing only "Browser sends JWT → API Gateway validates" skips how the browser acquired the JWT, which evaluators may probe.
**How to avoid:** Include the full Cognito Hosted UI redirect flow (authorization code with PKCE) or at minimum document the auth flow type (Authorization Code with PKCE) in the design notes.

### Pitfall 5: Missing the Comparison Table Audit
**What goes wrong:** D-14 requires auditing all phases to confirm every major decision has a comparison table. If Phase 3's ETL trigger table or query engine table is missing from the existing docs, the evaluator sees an incomplete assessment.
**How to avoid:** Before writing doc 10, explicitly verify that docs 01–07 contain tables for: IoT entry point, stream buffer, time-series store, relational store, ETL trigger, query engine. If any is missing, add a brief table to the overview doc with a note referencing the phase where the decision was made.

### Pitfall 6: OAI vs OAC in CloudFront S3 Config
**What goes wrong:** Writing OAI-style bucket policies or referencing the CloudFront origin identity in the bucket policy instead of the OAC service principal.
**Why it happens:** OAC was introduced in 2022; much training data and blog content still shows OAI.
**How to avoid:** Use the OAC bucket policy pattern explicitly (shown in Architecture Patterns section above). The key difference: OAC uses `"Service": "cloudfront.amazonaws.com"` with `"AWS:SourceArn"` condition; OAI uses `"CanonicalUser"` principal.

---

## Code Examples

### Cognito JWT Authorizer Configuration (API Gateway HTTP API)
```yaml
# Source: AWS API Gateway HTTP API documentation
# JWT authorizer — no Lambda needed
Auth:
  DefaultAuthorizer: CognitoAuthorizer
  Authorizers:
    CognitoAuthorizer:
      IdentitySource: $request.header.Authorization
      JwtConfiguration:
        Audience:
          - !Ref CognitoUserPoolClientId
        Issuer: !Sub https://cognito-idp.${AWS::Region}.amazonaws.com/${CognitoUserPoolId}
```
**Confidence:** HIGH — from AWS SAM documentation and API Gateway HTTP API docs.

### CloudFront OAC Bucket Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontOAC",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::iot-dashboard-spa/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EXAMPLEID"
        }
      }
    }
  ]
}
```
**Confidence:** HIGH — from AWS prescriptive guidance (Deploy React SPA to S3 and CloudFront).

### API Layer Mermaid Per-Layer Diagram Pattern
```mermaid
flowchart LR
    subgraph INTERNET["Internet"]
        Browser[Web Browser]
    end

    subgraph EDGE["AWS Edge"]
        CF[CloudFront\nWAF WebACL]
        APIGW[API Gateway\nHTTP API v2\nJWT Authorizer]
        COG[Cognito\nUser Pools]
    end

    subgraph VPC["VPC Private Subnet"]
        LAM[Lambda\nAPI Handlers\narm64 / Python 3.12]
        RDS_PROXY[RDS Proxy]
    end

    subgraph DATA["VPC Data Tier"]
        TS[(Timestream)]
        DDB[(DynamoDB)]
        AUR[(Aurora\nPostgreSQL\nServerless v2)]
    end

    Browser -->|HTTPS| CF
    CF -->|static assets\nOAC| S3[(S3 SPA Bucket)]
    CF -->|/api/* forward| APIGW
    Browser -->|auth redirect| COG
    COG -->|JWT| Browser
    APIGW -->|validate JWT| COG
    APIGW -->|VPC Link| LAM
    LAM -->|VPC endpoint| TS
    LAM -->|VPC endpoint| DDB
    LAM --> RDS_PROXY --> AUR
```

### Cost Table Structure (for 11-cost-analysis.md)
```markdown
| Service | 1,000 devices/month | 10,000 devices/month | Notes |
|---------|--------------------|--------------------|-------|
| IoT Core | $1–$3 | $10–$30 | With Basic Ingest |
| Kinesis Streams | $5–$10 | $20–$50 | On-Demand |
| Kinesis Firehose | $1–$3 | $10–$25 | |
| Lambda (processing) | $1–$5 | $5–$20 | Graviton2, batch 200 |
| Timestream | $1–$5 | $5–$25 | 1yr magnetic retention |
| DynamoDB | $1–$5 | $5–$20 | On-Demand |
| Aurora Serverless v2 | $10–$20 | $20–$50 | 0.5–4 ACU range |
| S3 + Glue + Athena | $3–$10 | $10–$30 | Parquet partitioned |
| API Gateway HTTP API | $1–$3 | $3–$10 | |
| CloudFront | $1–$3 | $3–$8 | SPA + API responses |
| Supporting services* | $15–$25 | $20–$35 | VPC endpoints, KMS, WAF, CloudWatch |
| **TOTAL** | **$40–$92** | **$111–$303** | |
```
*Supporting includes: WAF ($5/WebACL + $1/M req), VPC Interface endpoints (~3 endpoints × $7/AZ × 2 AZ = ~$42/month — largest fixed cost), KMS ($3–5), Secrets Manager ($2–4), CloudTrail (free management events).

**Important note on VPC endpoint costs:** Three Interface endpoints (Timestream, Kinesis, Secrets Manager) at $7/month × 2 AZs = ~$42/month fixed. This is the dominant cost at low scale — document explicitly as it surprises evaluators expecting "$0 for dev." Gateway endpoints (S3, DynamoDB) are free.

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| CloudFront OAI (Origin Access Identity) | CloudFront OAC (Origin Access Control) | Aug 2022 (GA) | OAC supports more S3 features (SSE-KMS, POST), is the recommended approach. OAI still works but is not recommended for new deployments. |
| API Gateway REST API v1 as default | API Gateway HTTP API v2 as default for Lambda | 2019 (HTTP API GA 2020) | HTTP API is 70% cheaper, lower latency, native JWT authorizer. REST API v1 retained only for advanced features (API keys, usage plans, SDK generation). |
| Cognito Identity Pools for web app auth | Cognito User Pools as primary; Identity Pools as optional for direct AWS access | Cognito User Pools is the long-standing standard | User Pools issue application JWTs; Identity Pools issue temporary IAM creds. Conflating them is a common architectural mistake. |
| Amazon IoT Analytics (deprecated) | AWS Glue + S3 + Athena | Apr 2023 (IoT Analytics deprecated) | Already handled in Phase 3 — relevant for DOC-03 comparison table audit to confirm this is not referenced. |

**Deprecated/outdated in this project context:**
- CloudFront OAI: replaced by OAC — do not reference OAI in Phase 4 docs.
- Amazon Kinesis Data Analytics (legacy SQL): deprecated in favor of Managed Apache Flink — already documented in CLAUDE.md "What NOT to Use" table, included in DOC-03 audit.
- Amazon IoT Analytics: deprecated April 2023 — already in CLAUDE.md "What NOT to Use" table.

---

## Open Questions

1. **Does the overview diagram reference doc 01–07 numbers explicitly?**
   - What we know: D-10 says overview is additive; per-layer diagrams retained unchanged.
   - What's unclear: Whether the overview diagram nodes should have hyperlinks or just descriptive labels referencing doc numbers in prose.
   - Recommendation: Use prose cross-references below each diagram (e.g., "See 02-device-connectivity-ingestion.md for IoT Core layer detail") rather than Mermaid hyperlinks — GitHub Mermaid rendering supports `click` hyperlinks but they require absolute URLs which would be brittle.

2. **Missing comparison tables audit — are ETL trigger and query engine tables already in doc 07?**
   - What we know: D-14 lists "ETL trigger (Phase 3)" and "query engine (Phase 3)" as required tables.
   - What's unclear: Whether these tables already appear in docs/architecture/07-data-lake-etl.md.
   - Recommendation: Planner should add a verification task at the start of the overview/sequences wave to read doc 07 and confirm ETL trigger (EventBridge cron vs S3-event-driven vs Glue Workflow) and query engine (Athena vs Redshift Spectrum vs EMR) tables exist. If absent, add them to doc 07 before writing the audit section in doc 10.

3. **Number of representative API endpoints per resource group**
   - What we know: D-02 says "representative endpoints" — this is Claude's discretion per the CONTEXT.md.
   - Recommendation: 2–3 endpoints per group (e.g., `GET /devices`, `GET /devices/{id}`, `POST /devices/{id}/commands` for the devices group). This demonstrates breadth without an exhaustive spec.

---

## Environment Availability

Step 2.6: SKIPPED — Phase 4 is a documentation-only task. No external tools, databases, CLI utilities, runtimes, or services need to be provisioned or checked. All deliverables are Markdown files with Mermaid diagram syntax.

---

## Project Constraints (from CLAUDE.md)

The following actionable directives from `./CLAUDE.md` constrain Phase 4:

| Directive | Impact on Phase 4 |
|-----------|-------------------|
| **Platform: AWS primary** | All services in docs 08–11 must be AWS-native. No third-party services as primary recommendations. |
| **Format: Mermaid diagrams — must be readable without tooling** | Diagrams must render correctly in standard GitHub Markdown. Test Mermaid syntax for common GitHub rendering issues (avoid spaces in node IDs, avoid special characters in labels without quoting). |
| **Documentation deliverable, not infrastructure code** | No Terraform, CDK, or CloudFormation snippets. Architecture description only. JSON policy snippets are acceptable as documentation examples. |
| **Device connectivity: hourly push, architecture must queue commands** | The web-to-device command flow sequence must explicitly show Device Shadow as the queuing mechanism — not direct MQTT publish to device topic. |
| **Security: Databases must be private (VPC-internal only)** | API layer doc must explicitly note Lambda runs in VPC private subnet and accesses all storage via VPC endpoints or RDS Proxy. No public endpoints. |
| **Scale target: Thousands of devices** | Cost analysis must cover 1,000–10,000 device range, not just a single point estimate. |
| **GSD workflow: no direct edits outside GSD** | Planner must structure tasks within GSD execute-phase workflow. |
| **Language: Match conversation language (Spanish/English)** | Document content in English (established by prior docs 01–07). |
| **Conventional Commits** | Git commits for Phase 4 docs must follow `docs(phase-4): ...` format. |

---

## Sources

### Primary (HIGH confidence)

- AWS API Gateway HTTP API documentation — JWT authorizer configuration, VPC Link pattern, HTTP API vs REST API pricing
- AWS CloudFront OAC documentation (Aug 2022 GA) — OAC bucket policy format, S3 integration
- AWS Prescriptive Guidance: Deploy React SPA to S3 and CloudFront — OAC bucket policy, CloudFront cache behavior
- AWS Cognito User Pools Developer Guide — Hosted UI, Authorization Code + PKCE flow, JWT claims
- CLAUDE.md (project file) — Full recommended stack, alternatives considered, cost optimization table, "What NOT to Use" table
- .planning/research/SUMMARY.md — Project research synthesis, phase ordering rationale, pitfalls
- docs/architecture/01–07 — Established diagram style, component names, cross-reference targets

### Secondary (MEDIUM confidence)

- AWS API Gateway Features comparison page — HTTP API vs REST API v1 feature matrix and pricing
- AWS blog: S3 + CloudFront vs Amplify Hosting comparison — hosting trade-offs
- CONTEXT.md canonical references — Cognito vs IAM Identity Center decision rationale

### Tertiary (LOW confidence — not needed, decisions already locked)

None. All decisions are locked in CONTEXT.md; no exploratory research was required.

---

## Metadata

**Confidence breakdown:**
- Document structure: HIGH — Four new files are fully specified in CONTEXT.md D-19/D-20
- API layer content: HIGH — Locked decisions in D-01 to D-05; patterns established in prior docs
- Web frontend content: HIGH — Locked decisions in D-06 to D-08; OAC pattern well-documented
- Overview diagram: HIGH (approach) / MEDIUM (exact Mermaid layout) — Claude's discretion
- Sequence diagrams: HIGH — Three diagrams fully specified in D-12/D-13; cross-references are clear
- Comparison tables: HIGH — Three new tables specified (D-14); existing tables in docs 01–07 well-established
- Cost analysis: HIGH — Pricing figures sourced from AWS pricing pages and CLAUDE.md research

**Research date:** 2026-03-28
**Valid until:** 2026-04-28 (stable — AWS pricing rarely changes; OAC is current standard)
