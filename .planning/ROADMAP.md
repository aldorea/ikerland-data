# Roadmap: IoT Cloud Architecture Design (AWS)

## Overview

This roadmap structures the architecture design document as four progressive documentation phases, each delivering a coherent, independently verifiable layer of the AWS IoT platform. Phase 1 establishes the security and connectivity foundation. Phase 2 documents the data pipeline from ingestion through storage and alarms. Phase 3 covers the Data Lake and ETL cold path. Phase 4 completes the document with the API, web frontend, and cross-cutting quality artifacts (diagrams, comparison tables, cost analysis). Every phase produces Mermaid diagrams and technology justifications as primary outputs — this is a documentation project, not an implementation.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [x] **Phase 1: Foundation, Device Connectivity & Security** - Document VPC/IAM/KMS security foundation and AWS IoT Core device layer (ingestion, certificates, Rules Engine, Device Shadow) (completed 2026-03-27)
- [ ] **Phase 2: Data Pipeline, Storage & Alarm Notifications** - Document the hot processing path (Kinesis → Lambda → Timestream/DynamoDB) and the full alarm notification pipeline (Rules Engine → SNS → EventBridge)
- [ ] **Phase 3: Data Lake & ETL** - Document the cold path S3 medallion Data Lake with Glue ETL, Parquet partitioning, and Athena query layer
- [ ] **Phase 4: API, Web Frontend & Documentation Quality** - Document the REST API (API Gateway + Cognito), SPA web frontend (S3 + CloudFront), and produce all cross-cutting quality artifacts (overview diagram, sequence diagrams, comparison tables, cost analysis)

## Phase Details

### Phase 1: Foundation, Device Connectivity & Security
**Goal**: The architecture document clearly describes the security foundation and device connectivity layer — evaluators can read exactly how VPC isolation, IAM roles, KMS encryption, X.509 certificates, IoT Core provisioning, topic routing, and Device Shadow command delivery work together
**Depends on**: Nothing (first phase)
**Requirements**: SEC-01, SEC-02, SEC-03, SEC-04, SEC-05, SEC-06, INGT-01, INGT-02, INGT-03, INGT-04, DEVM-01, DEVM-02, DEVM-03, DEVM-04
**Success Criteria** (what must be TRUE):
  1. A reader can identify every subnet tier (public/private/data) from the VPC diagram, including which services sit in which subnet and which VPC endpoints are Gateway vs Interface type
  2. A reader can trace the full device certificate lifecycle — from Fleet Provisioning claim certificate exchange through per-device X.509 issuance to an IoT policy scoped to `${iot:ThingName}` — without ambiguity
  3. A reader can see the complete topic namespace design and understand which Rules Engine SQL expression routes each message type (telemetry, alarm, config) to which downstream target
  4. A reader can follow the Device Shadow sequence diagram step-by-step: web writes desired state → Shadow stores → device reconnects → fetches delta → executes command → updates reported state → delta cleared
  5. The security section explicitly covers encryption at rest (KMS), encryption in transit (TLS 1.2+), IAM least-privilege roles per service, and WAF placement — with no hand-waving
**Plans**: 3 plans
Plans:
- [x] 01-01-PLAN.md — Security Foundation (VPC topology, IAM roles, KMS encryption, TLS, WAF)
- [x] 01-02-PLAN.md — Device Connectivity & Ingestion (IoT Core, topic namespace, Rules Engine, Kinesis)
- [x] 01-03-PLAN.md — Device Management (Device Shadow sequence, Fleet Provisioning, IoT policy, Thing Types/Groups)

### Phase 2: Data Pipeline, Storage & Alarm Notifications
**Goal**: The architecture document fully describes how telemetry flows from IoT Core through Kinesis into Timestream/DynamoDB (hot path), and how alarm events are detected, routed, deduplicated, and delivered to multiple notification channels
**Depends on**: Phase 1
**Requirements**: PROC-01, PROC-02, PROC-03, STOR-01, STOR-02, STOR-03, STOR-04, ALRM-01, ALRM-02, ALRM-03, ALRM-04
**Success Criteria** (what must be TRUE):
  1. A reader can see why Kinesis batch processing was chosen over per-message Lambda invocation, supported by a comparison table covering Kinesis vs direct Lambda vs SQS for the telemetry hot path
  2. A reader can identify all three storage tiers and their roles: Timestream (hot time-series with memory/magnetic retention configuration), DynamoDB (device metadata, latest-value cache, command queue), and Aurora Serverless v2 (users, roles, alert rules) — each with access restricted to VPC-internal services only
  3. A storage comparison table documents at least three alternatives (Timestream vs DynamoDB vs RDS/Aurora vs InfluxDB) with pros, cons, and a justified recommendation
  4. A reader can follow the alarm pipeline from IoT Rules Engine SQL threshold detection through Lambda evaluator (with deduplication strategy and DLQ) through SNS fan-out to SES email, and understand how EventBridge enables future notification channel extensibility
  5. SQS dead-letter queues are documented for all Lambda consumers — no silent data loss paths exist in the described architecture
**Plans**: 3 plans
Plans:
- [x] 01-01-PLAN.md — Security Foundation (VPC topology, IAM roles, KMS encryption, TLS, WAF)
- [x] 01-02-PLAN.md — Device Connectivity & Ingestion (IoT Core, topic namespace, Rules Engine, Kinesis)
- [ ] 01-03-PLAN.md — Device Management (Device Shadow sequence, Fleet Provisioning, IoT policy, Thing Types/Groups)

### Phase 3: Data Lake & ETL
**Goal**: The architecture document describes a complete, AI-ready Data Lake with medallion structure, Parquet partitioning, Glue ETL job, and Athena query access — demonstrating understanding of cost optimization through partitioning and format conversion
**Depends on**: Phase 2
**Requirements**: LAKE-01, LAKE-02, LAKE-03, LAKE-04, LAKE-05
**Success Criteria** (what must be TRUE):
  1. A reader can identify the three S3 medallion zones (Bronze: raw JSON, Silver: validated Parquet, Gold: aggregated/enriched) and understand which Glue ETL step transforms each zone
  2. A reader can see the Hive-style partition scheme (e.g., `year=/month=/day=/device_type=`) and a concrete example illustrating how this reduces Athena scan cost vs an unpartitioned layout
  3. The ETL trigger mechanism (periodic schedule vs event-driven via EventBridge) is documented with the chosen approach justified
  4. A reader can confirm that Athena queries the Silver/Gold Parquet layer — not raw JSON — and understands the workgroup configuration and cost controls in place
**Plans**: 3 plans
Plans:
- [x] 01-01-PLAN.md — Security Foundation (VPC topology, IAM roles, KMS encryption, TLS, WAF)
- [x] 01-02-PLAN.md — Device Connectivity & Ingestion (IoT Core, topic namespace, Rules Engine, Kinesis)
- [ ] 01-03-PLAN.md — Device Management (Device Shadow sequence, Fleet Provisioning, IoT policy, Thing Types/Groups)

### Phase 4: API, Web Frontend & Documentation Quality
**Goal**: The architecture document is complete: REST API and web frontend are fully described, and all cross-cutting quality artifacts (overview Mermaid diagram, per-layer diagrams, sequence diagrams for key flows, technology comparison tables, and cost analysis) are present and evaluator-ready
**Depends on**: Phase 3
**Requirements**: API-01, API-02, API-03, WEB-01, WEB-02, WEB-03, DOC-01, DOC-02, DOC-03, DOC-04, DOC-05
**Success Criteria** (what must be TRUE):
  1. A reader can trace a full API request: browser authenticates via Cognito (JWT), calls API Gateway HTTP API, Lambda handler in VPC private subnet queries Timestream or DynamoDB via VPC endpoint, response returned — with CloudFront and WAF protecting the SPA entry point
  2. A reader can follow the web-to-device command flow: UI button click → API Gateway → Lambda → IoT Device Shadow desired state → device reconnects and applies delta — referencing Phase 1 sequence diagram
  3. The document contains a top-level Mermaid diagram showing all architecture layers and data flows, plus individual per-layer diagrams for ingestion, processing, storage, alarm, Data Lake, API, and security
  4. Each major technology decision (IoT Core entry point, Kinesis vs alternatives, Timestream vs relational, Grafana vs SPA, API Gateway HTTP vs REST v1) has a comparison table with at least two alternatives, pros/cons, and a clear recommendation
  5. The cost analysis section provides an estimated monthly cost range for 1,000–10,000 devices at hourly telemetry, broken down by major service category, with cost optimization strategies identified
**UI hint**: yes
**Plans**: 3 plans
Plans:
- [ ] 01-01-PLAN.md — Security Foundation (VPC topology, IAM roles, KMS encryption, TLS, WAF)
- [ ] 01-02-PLAN.md — Device Connectivity & Ingestion (IoT Core, topic namespace, Rules Engine, Kinesis)
- [ ] 01-03-PLAN.md — Device Management (Device Shadow sequence, Fleet Provisioning, IoT policy, Thing Types/Groups)

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Foundation, Device Connectivity & Security | 3/3 | Complete   | 2026-03-27 |
| 2. Data Pipeline, Storage & Alarm Notifications | 0/TBD | Not started | - |
| 3. Data Lake & ETL | 0/TBD | Not started | - |
| 4. API, Web Frontend & Documentation Quality | 0/TBD | Not started | - |
