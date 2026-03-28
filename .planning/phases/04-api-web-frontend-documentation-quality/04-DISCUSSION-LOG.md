# Phase 4: API, Web Frontend & Documentation Quality - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-28
**Phase:** 04-api-web-frontend-documentation-quality
**Areas discussed:** API endpoint design, Web frontend approach, Overview diagram scope, Cost analysis depth
**Mode:** --auto (all decisions auto-selected as recommended defaults)

---

## API Endpoint Design

| Option | Description | Selected |
|--------|-------------|----------|
| Grouped by resource | Representative endpoints per resource domain (devices, telemetry, alarms, users) | :heavy_check_mark: |
| Full API spec | Exhaustive endpoint listing with request/response schemas | |
| High-level only | Just mention "REST API" without endpoint detail | |

**User's choice:** [auto] Grouped by resource — enough for evaluator to understand API surface
**Notes:** Auto-selected. Includes Cognito JWT sequence diagram, HTTP API v2 vs REST API v1 vs App Runner comparison table.

### Sub-decisions

| Question | Options | Selected |
|----------|---------|----------|
| Cognito auth flow documentation | Sequence diagram (recommended) / Prose description / Skip | Sequence diagram |
| API comparison table scope | HTTP API v2 vs REST v1 vs App Runner (recommended) / HTTP API vs REST only / Skip | Three-way comparison |
| Lambda VPC placement | VPC private subnet (recommended) / Public subnet / VPC Link only | VPC private subnet |
| API deployment documentation | Custom domain + ACM (recommended) / Default API Gateway URL | Custom domain + ACM |

---

## Web Frontend Approach

| Option | Description | Selected |
|--------|-------------|----------|
| SPA (S3+CloudFront) with comparison | Custom SPA recommended, Grafana and Amplify as alternatives in comparison table | :heavy_check_mark: |
| Managed Grafana primary | Grafana as main visualization, SPA as alternative | |
| SPA only (no comparison) | Document SPA without alternatives | |

**User's choice:** [auto] SPA with comparison table (SPA vs Grafana vs Amplify)
**Notes:** Auto-selected. CloudFront OAC with S3 bucket policy snippet, WAF integration, ACM certificate. Web-to-device command flow as new sequence diagram cross-referencing Phase 1 Device Shadow.

### Sub-decisions

| Question | Options | Selected |
|----------|---------|----------|
| CloudFront OAC depth | With S3 bucket policy snippet (recommended) / Architecture-level only | With snippet |
| Web-to-device command flow | New sequence diagram (recommended) / Prose cross-reference only | New sequence diagram |
| Cognito Identity Pools | Optional architectural note (recommended) / Primary pattern / Omit | Optional note |

---

## Overview Diagram Scope

| Option | Description | Selected |
|--------|-------------|----------|
| Single high-level overview + retain per-layer | One overview diagram showing all layers, per-layer diagrams unchanged | :heavy_check_mark: |
| Multiple overview diagrams | Split overview into 2-3 domain overviews | |
| Single comprehensive diagram | One massive diagram with all detail | |

**User's choice:** [auto] Single high-level overview + retain per-layer diagrams
**Notes:** Auto-selected. VPC and security as overlay annotations. Three sequence diagrams: telemetry ingestion, alarm notification, web-to-device command.

### Sub-decisions

| Question | Options | Selected |
|----------|---------|----------|
| Diagram complexity | Layer grouping with data flow arrows (recommended) / Full detail / Minimal | Layer grouping |
| Security in overview | Overlay annotations (recommended) / Separate boxes / Omit | Overlay annotations |
| Sequence diagram count | Three key flows (recommended) / All possible flows / One summary | Three key flows |

---

## Cost Analysis Depth

| Option | Description | Selected |
|--------|-------------|----------|
| Per-service table with range | Monthly range for 1K-10K devices, per service category | :heavy_check_mark: |
| Per-API-call granularity | Detailed unit pricing calculations | |
| High-level estimate only | Single total range without breakdown | |

**User's choice:** [auto] Per-service table with monthly range ($40-$145/month aligned with project research)
**Notes:** Auto-selected. Single production estimate, dev/staging note in prose. Top 5-7 optimization strategies with savings percentages.

### Sub-decisions

| Question | Options | Selected |
|----------|---------|----------|
| Scenario coverage | Single production + dev note (recommended) / Multi-scenario tables / Production only | Single + dev note |
| Optimization strategies | Dedicated subsection (recommended) / Inline notes / Separate document | Dedicated subsection |
| Alignment with CLAUDE.md | Align with $40-$145/month (recommended) / Independent calculation | Align |

---

## Claude's Discretion

- Exact Mermaid styling, color coding, and node arrangement
- Number of representative API endpoints per resource
- Cognito Hosted UI configuration detail level
- CloudFront cache policy specifics
- Whether to create a summary roll-up of all comparison tables
- Final document index layout

## Deferred Ideas

None — all discussion stayed within phase scope.
