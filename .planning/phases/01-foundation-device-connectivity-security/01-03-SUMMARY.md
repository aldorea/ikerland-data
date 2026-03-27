---
phase: 01-foundation-device-connectivity-security
plan: 03
subsystem: device-management
tags: [iot, device-shadow, fleet-provisioning, x509, iot-policy, thing-types, thing-groups, mermaid]
dependency_graph:
  requires: []
  provides:
    - docs/architecture/03-device-management.md
  affects:
    - Phase 04 (web-to-device command flow references Device Shadow section)
tech_stack:
  added: []
  patterns:
    - Device Shadow desired/reported/delta pattern for disconnected device command delivery
    - Fleet Provisioning by Claim for zero-touch device onboarding at scale
    - IoT policy ${iot:ThingName} variable scoping for per-device topic isolation
    - Thing Types and Thing Groups hierarchy for fleet organization and policy inheritance
key_files:
  created:
    - docs/architecture/03-device-management.md
  modified: []
decisions:
  - D-05 Fleet Provisioning by Claim selected — documented with full 10-step sequence and provisioning comparison table
  - D-06 IoT policies scoped with ${iot:ThingName} — documented with full policy JSON and anti-pattern warning
  - D-07 Certificate rotation marked as v2 consideration — documented in provisioning section
metrics:
  duration: 2 minutes
  completed: "2026-03-27"
  tasks_completed: 2
  files_created: 1
  files_modified: 0
---

# Phase 01 Plan 03: Device Management Documentation Summary

## One-liner

Device Shadow sequence diagram (6 actors, version-15 idempotency, OFFLINE rect block) plus Fleet Provisioning claim-cert exchange, `${iot:ThingName}`-scoped IoT policy, and Thing Types/Groups hierarchy for the IoT architecture document.

## What Was Built

`docs/architecture/03-device-management.md` — a complete Device Management section covering all 4 DEVM requirements:

1. **Device Shadow — Command Delivery (DEVM-01, DEVM-02):** Full Mermaid `sequenceDiagram` with 6 named actors (WebUI, APIGW, LambdaCmd, Shadow, Dev, Audit). Includes: `rect` block highlighting the OFFLINE period, `version: 15` in the delta message, `Note over Dev` explaining the idempotency guard (`version 15 > last applied (12)`), `activate/deactivate` blocks for device processing, both `PENDING` and `DELIVERED` audit statuses. Version-Based Idempotency subsection explains the optimistic locking mechanism. Comparison table (Shadow vs SQS vs DynamoDB polling).

2. **Fleet Provisioning (DEVM-03):** 10-step `sequenceDiagram` with Factory, Device, IoTCore, ProvisioningLambda (pre-hook), DynamoDB (allowlist) actors. Covers `CreateKeysAndCertificate` and `CreateCertificateFromCsr` options, `certificateOwnershipToken`, allowlist validation, Thing/cert/group creation. Certificate rotation noted as v2 (D-07). Provisioning comparison table (Fleet Provisioning, JITR, Manual).

3. **IoT Policy (DEVM-03):** Full JSON policy template with `${iot:ThingName}` scoping (7 occurrences). All 4 required actions: `iot:Connect`, `iot:Publish`, `iot:Subscribe`, `iot:Receive`. Security anti-pattern callout warning against `iot:*` wildcard.

4. **Thing Types and Thing Groups (DEVM-04):** `SensorDevice` and `GatewayDevice` Thing Types with attributes. `AllDevices` group hierarchy tree with TemperatureSensors (Factory-A, Factory-B subgroups), PressureSensors, Gateways. Policy inheritance explanation. Dynamic groups with `attributes.firmwareVersion < '2.0'` query example.

## Commits

| Task | Commit | Files | Description |
|------|--------|-------|-------------|
| Task 1 — Device Shadow section | c2284a1 | docs/architecture/03-device-management.md (created) | Full sequence diagram, concepts table, idempotency subsection, comparison table |
| Task 2 — Fleet Provisioning + policy + fleet org | 0fd9b25 | docs/architecture/03-device-management.md (modified) | Fleet Provisioning sequence, IoT policy JSON, Thing Types/Groups hierarchy |

## Deviations from Plan

None — plan executed exactly as written.

## Known Stubs

None — all content is complete architecture documentation. No placeholders, no TODO markers, no hardcoded empty values.
