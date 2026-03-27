# IoT Cloud Architecture Design (AWS)

## What This Is

A comprehensive cloud architecture design document for an IoT device monitoring platform on AWS. The deliverable is planning documentation — architecture diagrams (Mermaid), technology comparison tables with pros/cons, and justified recommendations — not infrastructure code. This is a technical assessment (prueba técnica) demonstrating cloud architecture expertise.

## Core Value

A well-documented, decoupled, scalable, and cost-effective AWS architecture that clearly explains every design decision with alternatives considered, enabling evaluators to assess cloud architecture competence.

## Requirements

### Validated

- ✓ IoT data ingestion: devices send telemetry, config, and alarm events (JSON) hourly + event-driven — Phase 1
- ✓ Database access restricted to internal AWS services only (no public access) — Phase 1
- ✓ Security components documented (encryption, IAM, VPC, etc.) — Phase 1
- ✓ Device command/configuration push via web (handling normally-disconnected devices) — Phase 1

### Active

- [ ] Scalable to thousands of devices
- [ ] Data processing pipeline: format, filter, and route messages by type
- [ ] Alarm notification system: email and other channels
- [ ] REST API deployed on the platform
- [ ] Web application for data visualization, deployed on the platform
- [ ] Periodic ETL to Data Lake in AI-ready format (Parquet/similar)
- [ ] Architecture diagrams with Mermaid showing all components and data flows
- [ ] Technology comparison tables: for each key decision, list alternatives with pros/cons and recommendation
- [ ] Cost optimization analysis
- [ ] Clear documentation of the complete architecture flow

### Out of Scope

- Infrastructure as Code implementation (Terraform, CDK, CloudFormation) — this is a design exercise, not implementation
- Actual AWS resource provisioning — no accounts or deployments
- Application source code — focus is architecture, not coding
- Multi-cloud implementation — AWS primary (may reference Azure/GCP equivalents as bonus)
- Detailed pricing calculations — high-level cost optimization principles only

## Context

- **Nature**: Technical assessment (prueba técnica) for a Cloud/AWS position at Ikerland
- **Domain**: IoT device monitoring — industrial/telemetry context
- **Deliverable**: Architecture design documents with diagrams, not working infrastructure
- **Key challenge**: Devices are normally disconnected, connecting only hourly to push data — command delivery must handle this pattern
- **Evaluation criteria**: AWS best practices, decoupled/scalable design, extensibility, maintainability, cost optimization, security, clear documentation
- **Bonus opportunity**: Optionally show equivalent design on Azure, GCP, or Kubernetes

## Constraints

- **Platform**: AWS primary — all services should be AWS-native
- **Format**: Documentation with Mermaid diagrams — must be readable without tooling
- **Device connectivity**: Devices connect hourly for push only — architecture must queue commands
- **Data format**: JSON ingestion, AI-ready format (Parquet) for Data Lake
- **Security**: Databases must be private (VPC-internal only)
- **Scale target**: Thousands of devices

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Documentation-only deliverable | Technical assessment focuses on design competence, not implementation | — Pending |
| AWS-first with optional multi-cloud notes | Primary requirement is AWS, bonus for showing cross-platform knowledge | — Pending |
| Mermaid for diagrams | Renders in Markdown, no external tools needed, version-controllable | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd:transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-03-27 after initialization*
