# Phase 3: Data Lake & ETL - Context

**Gathered:** 2026-03-27
**Status:** Ready for planning

<domain>
## Phase Boundary

Document the cold-path S3 Data Lake with medallion architecture (Bronze/Silver/Gold), AWS Glue ETL for JSON→Parquet transformation, Hive-style partitioning for cost optimization, and Amazon Athena as the query layer. Output is architecture documentation with Mermaid diagrams and technology comparison tables. This phase covers the analytical/batch path — complementing Phase 2's hot processing path.

</domain>

<decisions>
## Implementation Decisions

### Medallion Architecture
- **D-01:** Three-zone medallion Data Lake on S3:
  - **Bronze** (`s3://data-lake/bronze/telemetry/year=/month=/day=/`): Raw JSON as-is from Kinesis Data Firehose. Immutable landing zone — never modified after write. Serves as the audit trail and replay source.
  - **Silver** (`s3://data-lake/silver/telemetry/year=/month=/day=/device_type=/`): Schema-validated, cleaned Parquet. Transformations: null handling, timestamp normalization (UTC ISO-8601), JSON→Parquet conversion, field type enforcement, deduplication of late-arriving records.
  - **Gold** (`s3://data-lake/gold/`): Aggregated Parquet for analytics. Sub-zones: `hourly_metrics/` (avg, min, max per device per hour), `daily_rollups/` (per device-type fleet summary), `device_health/` (last-seen, uptime percentage). Pre-computed for dashboard and reporting queries.
- **D-02:** All three zones in the same S3 bucket with prefix-based separation. S3 Intelligent-Tiering on Bronze (auto-moves cold raw data to cheaper tiers after 30 days). Standard tier for Silver/Gold (frequently queried).
- **D-03:** Glue Data Catalog as the single metadata registry for all three zones. Each zone registered as a separate Glue database table (e.g., `datalake.bronze_telemetry`, `datalake.silver_telemetry`, `datalake.gold_hourly_metrics`).

### ETL Pipeline
- **D-04:** AWS Glue ETL jobs (serverless Spark) for all transformations. Bronze→Silver runs hourly via EventBridge Scheduler cron rule. Silver→Gold runs daily.
- **D-05:** ETL trigger comparison table: Scheduled (EventBridge cron) vs Event-driven (S3 event → Lambda → Glue StartJobRun) vs Glue Workflow. Scheduled selected as primary — simpler, predictable cost, sufficient for hourly device telemetry. Document event-driven as a low-latency alternative if near-real-time analytics are needed.
- **D-06:** Glue job configuration: Python Shell or PySpark (Glue 4.0), G.1X worker type, auto-scaling enabled. Include concrete transformation pseudo-code showing Bronze JSON → Silver Parquet conversion with schema enforcement.
- **D-07:** Glue Crawler runs after each ETL job to update the Data Catalog with new partitions. Alternative: explicit `ALTER TABLE ADD PARTITION` in the ETL job (faster, no crawler cost). Document both approaches with recommendation.

### Partitioning Strategy
- **D-08:** Hive-style partitioning: `year=/month=/day=/device_type=/` for Silver and Gold zones. Partition by `device_type` (not `device_id`) because thousands of device-specific partitions would create too many small files, degrading query performance and increasing S3 LIST call costs.
- **D-09:** Include concrete cost reduction example: "Querying one device type for one month scans ~2.5% of data vs 100% for unpartitioned layout. At $5/TB scanned, this turns a $5 query into a $0.13 query."
- **D-10:** Firehose dynamic partitioning writes Bronze data with Hive-style `year=/month=/day=/` prefixes (configured in Firehose delivery stream). The ETL job adds `device_type=` partitioning during Bronze→Silver transformation.

### Athena Query Layer
- **D-11:** Athena queries Silver and Gold Parquet — never raw Bronze JSON. Document why: Parquet columnar format enables predicate pushdown and column pruning, reducing scan volume 80-90%.
- **D-12:** Athena Workgroup configuration: `BytesScannedCutoffPerQuery` set to 1GB (prevents runaway queries), dedicated S3 results bucket (`s3://data-lake/athena-results/`), enforce workgroup settings (override client-side).
- **D-13:** Comparison table: Athena vs Redshift Spectrum vs EMR for Data Lake queries. Athena selected — serverless, pay-per-query, no cluster to manage. Include when to consider Redshift Spectrum (consistent high-volume queries) or EMR (complex ML/ETL pipelines).

### Integration with Phase 2
- **D-14:** The Data Lake cold path starts where Phase 2's Kinesis Data Firehose ends. Firehose delivers raw JSON to Bronze zone. Phase 3 documents everything from Bronze onward. Cross-reference Phase 2's `04-data-pipeline-processing.md` for the Firehose configuration.
- **D-15:** Lake Formation governs access control: column-level security, row filters for multi-team access. Document as the recommended access layer over raw S3 bucket policies.

### Documentation Format
- **D-16:** Continue established pattern: narrative → data flow Mermaid diagram → comparison table → design notes.
- **D-17:** Include a Mermaid diagram showing the complete cold path: Firehose → S3 Bronze → Glue ETL → S3 Silver → Glue ETL → S3 Gold → Athena. Include Glue Data Catalog as a cross-cutting component.
- **D-18:** Include a Mermaid diagram or table showing the partitioning scheme with a concrete S3 path example.

### Claude's Discretion
- Exact Glue worker type and number (G.1X vs G.2X, auto-scaling bounds)
- Glue Crawler vs explicit partition management detail level
- Athena query examples (simple SELECT vs complex aggregation)
- Whether to show Glue job Python pseudo-code or describe transformations narratively
- Lake Formation configuration depth (overview vs detailed policy examples)

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Project Context
- `.planning/PROJECT.md` — Project scope, constraints, evaluation criteria
- `.planning/REQUIREMENTS.md` — Full requirement list; Phase 3 maps: LAKE-01..05

### Prior Phase Output
- `docs/architecture/01-security-foundation.md` — VPC topology, S3 VPC Gateway endpoint, KMS encryption (Phase 3 inherits S3 security)
- `docs/architecture/04-data-pipeline-processing.md` — Kinesis Data Firehose configuration, Lambda processing (Phase 3 continues from Firehose → S3)
- `docs/architecture/05-storage-layer.md` — DynamoDB and Timestream documentation (Phase 3 complements as the cold-path counterpart)

### Prior Phase Context
- `.planning/phases/01-foundation-device-connectivity-security/01-CONTEXT.md` — VPC decisions, KMS, S3 endpoint
- `.planning/phases/02-data-pipeline-storage-alarm-notifications/02-CONTEXT.md` — Firehose configuration, storage tiers, documentation format

### Research
- `.planning/research/STACK.md` — AWS service recommendations (Glue, Athena, Lake Formation details)
- `.planning/research/ARCHITECTURE.md` — Data Lake patterns, medallion architecture
- `.planning/research/PITFALLS.md` — ETL and partitioning gotchas

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `docs/architecture/04-data-pipeline-processing.md` — Documents Firehose delivery to S3. Phase 3's Bronze zone documentation must reference this as the data source.
- `docs/architecture/05-storage-layer.md` — Storage comparison table format. Phase 3's Athena comparison table should follow the same structure.
- Established Mermaid diagram style (`flowchart LR`) and comparison table format from Phase 1-2 docs.

### Established Patterns
- Architecture doc sections follow: narrative introduction → component inventory/table → Mermaid diagram → comparison table (if applicable) → design notes/anti-patterns
- Comparison tables use: Alternative | Pros | Cons | Cost | Recommendation columns
- Phase cross-references use explicit file paths (e.g., "see `01-security-foundation.md` for VPC topology")

### Integration Points
- Phase 3 Data Lake starts where Phase 2 Firehose delivery ends — `s3://data-lake/bronze/` prefix
- S3 encryption via KMS (Phase 1 SEC-04) applies to all Data Lake zones
- VPC Gateway endpoint for S3 (Phase 1 D-04) enables private access from Glue jobs running in VPC
- Glue Data Catalog will be referenced by Phase 4 if API needs to expose historical data via Athena

</code_context>

<specifics>
## Specific Ideas

- Success criteria #2 explicitly requires a concrete cost reduction example with Hive-style partitioning — must include actual dollar amounts, not just percentages
- Success criteria #1 requires the three medallion zones to be clearly identifiable with which ETL step transforms each — the diagram must make this traceable
- Success criteria #3 explicitly requires ETL trigger mechanism justification — the comparison table is mandatory, not optional
- The Firehose → Bronze connection is the critical Phase 2 → Phase 3 handoff point — must be unambiguous

</specifics>

<deferred>
## Deferred Ideas

- Apache Iceberg table format for ACID transactions and time-travel queries — v2 enhancement when Data Lake matures
- Amazon Managed Grafana connected to Athena for Data Lake visualization — Phase 4 consideration
- Real-time ETL via Managed Service for Apache Flink — not needed for hourly device data
- Data quality framework (Great Expectations or AWS Deequ) — v2 quality layer
- Cross-account Data Lake sharing via Lake Formation — v2 multi-team feature

</deferred>

---

*Phase: 03-data-lake-etl*
*Context gathered: 2026-03-27*
