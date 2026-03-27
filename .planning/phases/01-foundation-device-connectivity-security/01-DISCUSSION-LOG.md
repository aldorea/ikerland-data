# Phase 1: Foundation, Device Connectivity & Security - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-27
**Phase:** 01-Foundation, Device Connectivity & Security
**Areas discussed:** VPC topology design, Device provisioning model, Topic namespace strategy, Documentation format & structure
**Mode:** Auto (all recommended defaults selected)

---

## VPC Topology Design

| Option | Description | Selected |
|--------|-------------|----------|
| 3-tier (public/private/data) | Standard AWS Well-Architected pattern with full subnet isolation | ✓ |
| 2-tier (public/private) | Simpler but databases share subnet with compute | |
| Single private subnet | Minimal but no isolation between services | |

**User's choice:** [auto] 3-tier (public/private/data) — recommended default
**Notes:** Multi-AZ (2 AZs) selected. Single NAT Gateway for cost optimization.

---

## Device Provisioning Model

| Option | Description | Selected |
|--------|-------------|----------|
| Fleet Provisioning with claim cert | Scalable, automated, AWS-recommended for fleets | ✓ |
| Manual per-device registration | Simple but doesn't scale to thousands | |
| Just-in-Time Registration (JITR) | Requires CA integration, more complex | |

**User's choice:** [auto] Fleet Provisioning with claim certificate — recommended default
**Notes:** Certificate rotation documented as v2 consideration.

---

## Topic Namespace Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| devices/{thingName}/{messageType} | Clean hierarchy, enables topic-based routing | ✓ |
| {messageType}/devices/{thingName} | Groups by type first, harder to scope per-device | |
| Flat namespace with attributes | Uses message body for routing, less efficient | |

**User's choice:** [auto] devices/{thingName}/{messageType} — recommended default
**Notes:** One Rules Engine SQL rule per message type. errorAction to SQS DLQ on all rules.

---

## Documentation Format & Structure

| Option | Description | Selected |
|--------|-------------|----------|
| Single markdown with Mermaid | Version-controllable, renders in GitHub, comprehensive | ✓ |
| Multiple docs per layer | More modular but harder to read as a whole | |
| Presentation format (slides) | Evaluator-friendly but less detailed | |

**User's choice:** [auto] Single comprehensive markdown with Mermaid — recommended default
**Notes:** Inline comparison tables per section. Security as cross-cutting concern.

---

## Claude's Discretion

- Mermaid diagram styling and layout
- IAM policy example detail level
- Whether to include further reading links

## Deferred Ideas

None — auto mode stayed within phase scope
