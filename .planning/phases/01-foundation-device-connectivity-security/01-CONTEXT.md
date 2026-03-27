# Phase 1: Foundation, Device Connectivity & Security - Context

**Gathered:** 2026-03-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Document the complete security foundation (VPC, IAM, KMS, WAF) and device connectivity layer (IoT Core, X.509 certificates, Rules Engine, Device Shadow) for an AWS IoT monitoring platform. Output is architecture documentation with Mermaid diagrams and technology comparison tables — not infrastructure code.

</domain>

<decisions>
## Implementation Decisions

### VPC Topology
- **D-01:** 3-tier VPC design: public subnet (NAT Gateway, ALB), private subnet (Lambda, API Gateway VPC Link), data subnet (Timestream VPC endpoint, DynamoDB VPC endpoint, Aurora). All databases exclusively in data subnet with no public access.
- **D-02:** Multi-AZ deployment across 2 Availability Zones for resilience.
- **D-03:** Single NAT Gateway for cost optimization (acceptable for this scale; note multi-NAT for production HA in the doc as an option).
- **D-04:** VPC endpoints: Gateway type for S3 and DynamoDB (free), Interface type for Timestream, IoT Core credential provider, and other services.

### Device Provisioning
- **D-05:** Fleet Provisioning with claim certificate model — devices exchange a shared claim cert for unique per-device X.509 certificates on first connection.
- **D-06:** IoT policies scoped with `${iot:ThingName}` variable — each device can only publish/subscribe to its own topic namespace.
- **D-07:** Certificate rotation strategy documented as a design consideration but marked as v2 implementation detail.

### Topic Namespace
- **D-08:** Topic hierarchy: `devices/{thingName}/telemetry`, `devices/{thingName}/alarm`, `devices/{thingName}/config`, `$aws/things/{thingName}/shadow/...` (AWS managed).
- **D-09:** IoT Rules Engine uses topic-based SQL routing with one rule per message type, each routing to its appropriate downstream target (Kinesis for telemetry, Lambda for alarms, etc.).
- **D-10:** errorAction configured on all rules routing to SQS DLQ for failed deliveries.

### Documentation Format
- **D-11:** Single comprehensive markdown document with one section per architectural layer. Mermaid.js for all diagrams (renders natively in GitHub/GitLab).
- **D-12:** Comparison tables inline within each section — for each decision point, a table with alternatives, pros, cons, and clear recommendation with rationale.
- **D-13:** Security section structured as a cross-cutting concern: VPC diagram first, then IAM roles inventory, then encryption (KMS at rest, TLS in transit), then WAF placement.

### Claude's Discretion
- Exact Mermaid diagram styling and layout choices
- Level of detail in IAM policy examples (pseudo-policy vs full JSON)
- Whether to include a "further reading" links section

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Project scope, constraints, evaluation criteria
- `.planning/REQUIREMENTS.md` — Full requirement list with phase mapping (SEC-01..06, INGT-01..04, DEVM-01..04 for this phase)

### Research
- `.planning/research/STACK.md` — AWS service recommendations with alternatives and rationale
- `.planning/research/ARCHITECTURE.md` — Component boundaries, data flows, architectural patterns
- `.planning/research/PITFALLS.md` — Critical mistakes to avoid (Device Shadow, IoT policy variables, VPC endpoints)
- `.planning/research/SUMMARY.md` — Synthesized research findings

No external specs — requirements fully captured in decisions above and research files.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None — greenfield project, no existing codebase

### Established Patterns
- None — first phase, patterns will be established here

### Integration Points
- This phase's VPC and security documentation will be referenced by all subsequent phases
- Topic namespace design constrains Phase 2 (processing pipeline) and Phase 3 (Data Lake ETL triggers)
- Device Shadow documentation will be referenced by Phase 4 (web-to-device command flow)

</code_context>

<specifics>
## Specific Ideas

- Evaluators are looking for specific AWS knowledge — include IoT policy variable `${iot:ThingName}` examples, not just mention them
- The Device Shadow sequence diagram is the most evaluator-scrutinized flow — make it detailed and step-by-step
- VPC endpoint types (Gateway vs Interface) must be explicitly labeled — this is an evaluator litmus test
- Fleet Provisioning claim certificate exchange is the recommended provisioning model for this scale

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope

</deferred>

---

*Phase: 01-foundation-device-connectivity-security*
*Context gathered: 2026-03-27*
