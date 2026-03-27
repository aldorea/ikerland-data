# Phase 3: Data Lake & ETL - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-03-27
**Phase:** 03-data-lake-etl
**Areas discussed:** ETL trigger mechanism, Medallion zone transformation boundaries, Partitioning strategy, Athena cost controls, Firehose-to-Bronze integration
**Mode:** --auto (all selections made by recommended defaults)

---

## ETL Trigger Mechanism

| Option | Description | Selected |
|--------|-------------|----------|
| Periodic schedule (EventBridge cron) + event-driven documented | Hourly Bronze→Silver, daily Silver→Gold via EventBridge Scheduler. Event-driven as documented alternative. | [auto] |
| Event-driven only (S3 event → Lambda → Glue) | Near-real-time ETL triggered by new S3 objects | |
| Glue Workflow orchestration | Native Glue Workflows for multi-step ETL | |

**User's choice:** [auto] Periodic schedule + event-driven documented (recommended default)
**Notes:** Success criteria #3 requires justification. Schedule is simpler, predictable cost, sufficient for hourly device data. Event-driven adds complexity for near-real-time that isn't needed.

---

## Medallion Zone Transformation Boundaries

| Option | Description | Selected |
|--------|-------------|----------|
| Clear three-tier with concrete examples | Bronze=raw JSON, Silver=validated Parquet, Gold=aggregated. Concrete transformation example per tier. | [auto] |
| High-level description only | Describe tiers conceptually without transformation details | |
| Two-tier (skip Gold) | Only Bronze and Silver, aggregations done at query time | |

**User's choice:** [auto] Clear three-tier with concrete examples (recommended default)
**Notes:** Success criteria #1 requires identifiable zones with ETL step traceability. Gold tier demonstrates analytical value beyond simple storage.

---

## Partitioning Strategy

| Option | Description | Selected |
|--------|-------------|----------|
| year/month/day/device_type | Hive-style, device_type avoids small-file problem | [auto] |
| year/month/day/device_id | Per-device partitions — too many small files at thousands of devices | |
| year/month/day only | Simpler but less query optimization | |

**User's choice:** [auto] year/month/day/device_type (recommended default)
**Notes:** Success criteria #2 requires the scheme plus a concrete cost example. device_type partitioning is the right granularity for thousands of devices.

---

## Athena Cost Controls

| Option | Description | Selected |
|--------|-------------|----------|
| Workgroup with scan limit | BytesScannedCutoffPerQuery, dedicated results bucket, enforced settings | [auto] |
| Basic Athena setup | Default workgroup, no limits | |
| Redshift Spectrum instead | Dedicated cluster for consistent query performance | |

**User's choice:** [auto] Workgroup with scan limit (recommended default)
**Notes:** Athena is the right choice for serverless ad-hoc queries. Workgroup config shows production-readiness awareness.

---

## Firehose-to-Bronze Integration

| Option | Description | Selected |
|--------|-------------|----------|
| Explicit S3 path convention + Glue Data Catalog | Firehose → S3 bronze with Hive prefixes, Glue Crawler/Catalog registration | [auto] |
| Implicit (just reference Phase 2) | Assume reader understands the connection | |
| Duplicate Firehose config in Phase 3 doc | Repeat the Firehose configuration details | |

**User's choice:** [auto] Explicit S3 path convention + Glue Data Catalog (recommended default)
**Notes:** Critical handoff between Phase 2 and Phase 3. Must be explicit — cross-reference Phase 2 doc, don't duplicate.

---

## Claude's Discretion

- Glue worker type and auto-scaling bounds
- Crawler vs explicit partition management detail
- Athena query example complexity
- Transformation pseudo-code vs narrative
- Lake Formation configuration depth

## Deferred Ideas

- Apache Iceberg table format → v2
- Managed Grafana + Athena → Phase 4
- Real-time ETL via Flink → not needed for hourly data
- Data quality framework → v2
- Cross-account Lake Formation → v2
