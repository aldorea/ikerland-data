# Phase 4: API, Web Frontend & Documentation Quality - Context

**Gathered:** 2026-03-28
**Status:** Ready for planning

<domain>
## Phase Boundary

Document the REST API layer (API Gateway HTTP API + Lambda + Cognito authentication), SPA web frontend hosting (S3 + CloudFront with OAC), web-to-device command flow, and produce all cross-cutting quality artifacts: top-level overview Mermaid diagram, per-layer diagrams, sequence diagrams for key flows, technology comparison tables for remaining decisions, and cost analysis with per-service breakdown. Output is architecture documentation completing the deliverable — not infrastructure code.

</domain>

<decisions>
## Implementation Decisions

### API Layer
- **D-01:** Document API Gateway HTTP API (v2) + Lambda handlers as the REST API pattern. Lambda handlers run in VPC private subnet for direct access to Timestream, DynamoDB, and Aurora via VPC endpoints. RDS Proxy mediates Lambda → Aurora connections (inheriting Phase 2 decision).
- **D-02:** Endpoints documented grouped by resource domain (devices, telemetry, alarms, users/auth) with representative endpoints per group — not an exhaustive API spec. Enough detail for an evaluator to understand the API surface and access patterns.
- **D-03:** Cognito User Pools for user authentication with JWT-based authorization. Include sequence diagram: browser → Cognito Hosted UI → JWT issued → API Gateway JWT authorizer validates token → Lambda handler executes. Cognito Identity Pools documented as optional for direct frontend→AWS service access (e.g., Timestream queries), but API-mediated access recommended as the primary pattern for security and auditability.
- **D-04:** API comparison table: API Gateway HTTP API v2 vs REST API v1 vs App Runner. Evaluation axes: cost, features (JWT auth, VPC Link, usage plans), latency, cold start. HTTP API v2 recommended (70% cheaper, native JWT authorizer, sufficient for this platform).
- **D-05:** API deployed within AWS platform — API Gateway is AWS-managed, Lambda runs in VPC. No external hosting. Document the API Gateway custom domain + ACM certificate pattern.

### Web Frontend
- **D-06:** Comparison table: SPA on S3+CloudFront vs Amazon Managed Grafana vs AWS Amplify Hosting. SPA recommended for custom IoT dashboards requiring device command UI. Grafana noted as simpler for operations-only monitoring teams. Amplify noted as a developer-friendly wrapper around S3+CloudFront.
- **D-07:** CloudFront Origin Access Control (OAC) documented with S3 bucket policy snippet restricting access to CloudFront distribution only. ACM certificate for HTTPS on custom domain. WAF attached to CloudFront distribution (inheriting Phase 1 SEC-06).
- **D-08:** Web-to-device command flow: new sequence diagram showing UI button click → API Gateway → Lambda → IoT Core Device Shadow `desired` state update → cross-reference Phase 1 Device Shadow delta sequence (device reconnects → receives delta → applies → updates reported). Single end-to-end flow that ties Phase 1 and Phase 4 together.

### Overview Diagram
- **D-09:** One high-level overview Mermaid diagram showing all architecture layers (IoT Ingestion → Processing Pipeline → Storage → Data Lake → API → Web Frontend → Notifications) with simplified service icons and data flow arrows between layers. VPC boundary and security elements (KMS, WAF) shown as overlay annotations, not separate boxes.
- **D-10:** All existing per-layer diagrams (from docs 01–07) retained unchanged. The overview diagram is additive — it provides the 30,000-foot view while per-layer diagrams provide depth.
- **D-11:** Per-layer diagrams for API and Web Frontend layers added as new content in this phase's architecture docs.

### Sequence Diagrams
- **D-12:** Three key-flow sequence diagrams:
  1. **Telemetry ingestion end-to-end:** Device → IoT Core → Rules Engine → Kinesis → Lambda → Timestream + DynamoDB (references docs 02, 04, 05)
  2. **Alarm notification pipeline:** Device → IoT Core → Lambda evaluator → DynamoDB dedup → SNS → SES email (references docs 02, 06)
  3. **Web-to-device command delivery:** Browser → Cognito → API Gateway → Lambda → IoT Device Shadow → device reconnect → delta apply (references docs 03, new Phase 4 API doc)
- **D-13:** Sequence diagrams use Mermaid `sequenceDiagram` syntax. Each diagram cross-references the detailed section where each component is documented.

### Technology Comparison Tables
- **D-14:** Ensure every major technology decision has a comparison table. Audit across all phases:
  - IoT entry point: IoT Core vs SiteWise vs self-managed MQTT (Phase 1)
  - Stream buffer: Kinesis vs MSK vs SQS (Phase 2)
  - Time-series store: Timestream vs DynamoDB vs InfluxDB (Phase 2)
  - Relational store: Aurora Serverless v2 vs RDS Provisioned vs DynamoDB (Phase 2)
  - ETL trigger: EventBridge cron vs S3-event-driven vs Glue Workflow (Phase 3)
  - Query engine: Athena vs Redshift Spectrum vs EMR (Phase 3)
  - **NEW in Phase 4:** API front door (HTTP API v2 vs REST API v1 vs App Runner)
  - **NEW in Phase 4:** Web hosting (SPA S3+CloudFront vs Grafana vs Amplify)
  - **NEW in Phase 4:** Auth provider (Cognito User Pools vs IAM Identity Center vs self-managed)
- **D-15:** Each comparison table uses consistent format: Alternative | Pros | Cons | Recommendation. Tables already in prior docs are referenced, not duplicated.

### Cost Analysis
- **D-16:** Per-service-category cost table with monthly range for 1,000–10,000 devices at hourly telemetry. Categories: IoT Core, Kinesis (Streams + Firehose), Lambda, Timestream, DynamoDB, Aurora Serverless v2, S3 + Glue + Athena, API Gateway, CloudFront, supporting services (CloudWatch, KMS, VPC endpoints, Secrets Manager).
- **D-17:** Single production estimate with low–high range. Prose note that dev/staging costs approximately 10–20% of production (Aurora scale-to-zero, reduced throughput, no multi-AZ). Align with the $40–$145/month range established in project research for thousands of devices.
- **D-18:** Dedicated cost optimization subsection listing top 5–7 strategies with estimated savings: IoT Core Basic Ingest (~50% messaging cost), Graviton2 Lambda (~20% cheaper), Parquet + partitioning (~80–90% Athena scan reduction), S3 Intelligent-Tiering, DynamoDB On-Demand + TTL, Aurora Serverless v2 scale-to-zero, CloudFront compression.

### Documentation Format
- **D-19:** Continue established pattern from Phases 1–3: narrative → Mermaid diagram → comparison table → design notes. Phase 4 adds the overview diagram and sequence diagrams as new document types.
- **D-20:** Final architecture document structure: one file per architectural layer (01–07 from prior phases, plus new 08-api-layer.md, 09-web-frontend.md, 10-overview-and-sequences.md, 11-cost-analysis.md), with a table of contents or index file.

### Claude's Discretion
- Exact Mermaid styling, color coding, and node arrangement in overview diagram
- Level of detail in API endpoint listing (number of representative endpoints per resource)
- Whether to include Cognito Hosted UI configuration details or keep at architectural level
- CloudFront cache policy specifics (TTL values, behavior patterns)
- Whether comparison tables from prior phases need a summary roll-up table in the cost section
- Exact layout of the final document index

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Project scope, constraints, evaluation criteria
- `.planning/REQUIREMENTS.md` — Full requirement list; Phase 4 maps: API-01..03, WEB-01..03, DOC-01..05

### Prior Phase Output (Architecture Docs)
- `docs/architecture/01-security-foundation.md` — VPC topology, IAM, KMS, WAF (Phase 4 API/web inherit security)
- `docs/architecture/02-device-connectivity-ingestion.md` — IoT Core, topic namespace, Rules Engine (referenced in overview diagram and telemetry sequence)
- `docs/architecture/03-device-management.md` — Device Shadow sequence (referenced in web-to-device command flow)
- `docs/architecture/04-data-pipeline-processing.md` — Kinesis, Lambda processing (referenced in telemetry sequence)
- `docs/architecture/05-storage-layer.md` — Timestream, DynamoDB, Aurora (API Lambda queries these)
- `docs/architecture/06-alarm-notifications.md` — SNS, EventBridge, SES (referenced in alarm sequence)
- `docs/architecture/07-data-lake-etl.md` — Medallion Data Lake, Glue, Athena (referenced in overview diagram)

### Prior Phase Context
- `.planning/phases/01-foundation-device-connectivity-security/01-CONTEXT.md` — VPC decisions, security patterns, doc format
- `.planning/phases/02-data-pipeline-storage-alarm-notifications/02-CONTEXT.md` — Processing pipeline, storage tiers, alarm pipeline, doc format
- `.planning/phases/03-data-lake-etl/03-CONTEXT.md` — Data Lake, ETL, partitioning, Athena, doc format

### Research
- `.planning/research/STACK.md` — AWS service recommendations with alternatives (API Gateway, Cognito, CloudFront)
- `.planning/research/ARCHITECTURE.md` — Component boundaries, data flows
- `.planning/research/PITFALLS.md` — Critical mistakes to avoid
- `.planning/research/SUMMARY.md` — Synthesized research findings

### CLAUDE.md Stack Reference
- `CLAUDE.md` — Contains the full recommended technology stack with alternatives considered, cost optimization table, and sources. Phase 4 should align with these recommendations.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- 7 architecture documents (01–07) with established structure: narrative → Mermaid → comparison table → design notes
- Mermaid diagram patterns established across all 7 docs (consistent style, service naming)
- Comparison table format consistent: Alternative | Pros | Cons | Recommendation

### Established Patterns
- Each architecture doc is a standalone markdown section with cross-references to related docs
- Mermaid diagrams use `graph TD` or `graph LR` for architecture, `sequenceDiagram` for flows
- VPC topology and security annotations consistent across all docs
- Phase 1 Device Shadow sequence diagram provides the template for Phase 4 command flow diagram

### Integration Points
- API Lambda handlers connect to Timestream (doc 05), DynamoDB (doc 05), Aurora (doc 05) — all via VPC endpoints (doc 01)
- Web-to-device command flow connects UI (new) → API (new) → Device Shadow (doc 03)
- Overview diagram must reference all 7 existing docs plus 3–4 new docs
- Cost analysis draws numbers from all layers documented in prior phases

</code_context>

<specifics>
## Specific Ideas

No specific requirements — open to standard approaches. Prior phases established a clear documentation pattern that Phase 4 should continue. The key challenge is producing a cohesive overview that ties all layers together while adding the final API and web frontend documentation.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope.

v2 bonus items from REQUIREMENTS.md remain deferred:
- MCLD-01/02/03: Multi-cloud equivalents (Azure, GCP, Kubernetes)
- ADV-01..05: Device Defender, Greengrass, CI/CD, DR/multi-region

</deferred>

---

*Phase: 04-api-web-frontend-documentation-quality*
*Context gathered: 2026-03-28*
