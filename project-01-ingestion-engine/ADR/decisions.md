# Architecture Decision Records

This file captures all architectural and technical decisions made for **Project 1: Schema-on-Read Ingestion Engine**.

Every decision — no matter how small — that shapes the architecture, data flow, tool selection, or operational behavior of this system must be recorded here. This prevents silent decisions from accumulating into unexplained technical debt.

---

## ADR-001: Ingestion Entry Point — API Gateway REST Endpoint

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
Source systems need a way to deliver JSON and CSV payloads into the raw S3 zone. Three options were evaluated. The entry point determines latency characteristics, backpressure handling, and how much coupling exists between sources and the pipeline.

**Decision:**
Use **API Gateway (REST API) → Lambda → S3 PutObject** as the ingestion entry point.

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| API Gateway (chosen) | Standard HTTP interface, easy for any source to call, request validation possible, auth via IAM or API keys | Lambda payload limit (6MB sync / 10MB async), not suited for very large files |
| Direct S3 Upload (presigned URL) | No size limits, no Lambda in the hot path, simple for bulk file drops | No standard interface, sources must implement presigned URL flow, harder to enforce source metadata |
| Kinesis Firehose | High-throughput streaming, built-in buffering, native S3 delivery | Over-engineered for batch/file-based sources, adds cost, streaming semantics mismatch for CSV files |

**Consequences:**
- Sources interact over standard HTTPS — easy to integrate, no SDK required.
- Payload size is bounded. Files > 6MB must use multipart upload or a separate presigned URL flow. This boundary must be documented for source teams.
- API Gateway provides a natural place to enforce authentication (API key or IAM SigV4) before data enters the pipeline.
- A 4xx/5xx response to the source gives immediate feedback — unlike fire-and-forget S3 writes.

---

## ADR-002: Schema Profiling Trigger — S3 Event Notification → Lambda

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
After a raw file lands in S3, the system must profile its structure and write a schema registry entry to DynamoDB. The trigger mechanism determines how quickly profiling happens and how reliably every file is processed.

**Decision:**
Use **S3 Event Notifications → Lambda** to trigger the schema profiler on every `s3:ObjectCreated:*` event in the raw zone prefix.

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| S3 Event Notification → Lambda (chosen) | Near-real-time, per-object granularity, low operational overhead | Lambda concurrency limits under high write bursts; cold start latency |
| EventBridge Pipe (S3 → EventBridge → Lambda) | More routing flexibility, event filtering, dead-letter support built in | Added latency (seconds), more infrastructure to maintain, slight cost increase |
| Scheduled polling | Simple, predictable | Latency up to poll interval; race conditions between ingestion and profiling |

**Consequences:**
- Every file that lands in the raw zone triggers a profiling Lambda within seconds.
- Lambda must handle concurrent invocations. DynamoDB writes must use conditional expressions to prevent race conditions on schema version creation.
- Files that fail profiling (malformed, unreadable) must be moved to `s3://raw-zone/dead-letter/` by the Lambda and a CloudWatch metric must be emitted. This is the primary error path.

---

## ADR-003: Schema Registry — Custom DynamoDB Table vs. AWS Glue Data Catalog

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
AWS Glue Data Catalog is the standard AWS schema registry for data lakes and integrates natively with Athena, EMR, and Glue jobs. However, Glue Catalog treats schemas as table-level definitions, not versioned field-level records. This project requires tracking field-level diffs, rename detection, and compatibility flags — which Glue Catalog does not natively provide.

**Decision:**
Build a **custom schema registry in DynamoDB** with the following key design:

- **PK:** `source_id` (String) — identifies the originating system
- **SK:** `version#<ISO-timestamp>` (String) — enables chronological version queries
- **`fields`** (Map): `{ field_name: inferred_type }` for the full payload schema
- **`field_diff`** (Map): `{ added: [...], removed: [...], renamed: { old: new } }`
- **`status`** (String): `ACTIVE` | `DEPRECATED` | `BREAKING`
- **`compatibility`** (String): `BACKWARD` | `FORWARD` | `FULL` | `NONE`
- **`promoted_at`** (String, ISO timestamp): last successful promotion timestamp

**Versioning Strategy:** Union-by-default. New fields in an incoming payload are additive — they extend the schema and are non-breaking. A field present in the prior `ACTIVE` version that is absent in the new payload is flagged as `BREAKING`. No version is ever deleted; old versions are `DEPRECATED`.

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| Custom DynamoDB (chosen) | Full control over versioning model, field-level diffs, rename detection, compatibility flags | Must build and maintain; not natively understood by Athena/EMR |
| AWS Glue Data Catalog | Native AWS integration, Athena and EMR read it directly, no custom code | Table-level only, no field-level diff history, no rename detection, no compatibility semantics |
| AWS Glue Schema Registry (separate service) | Designed for schema versioning, supports Avro/JSON Schema/Protobuf, compatibility modes | Requires schemas to be submitted explicitly (not inferred), does not auto-profile arriving payloads |

**Consequences:**
- The curated zone Parquet files must still be registered in Glue Catalog (for Athena access). The custom DynamoDB registry and the Glue Catalog serve different purposes and coexist.
- The promotion job must read from DynamoDB (for validation/diff logic) and write to Glue Catalog (for query access). This is intentional separation of concerns.
- Rename detection (`field_diff.renamed`) is heuristic — inferred by matching removed fields to added fields of the same type. This will produce false positives. The `BREAKING` status requires human review before promotion can proceed.

---

## ADR-004: Promotion Trigger — EventBridge Scheduled Rule

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
The promotion step (raw → curated, Parquet conversion via EMR) must be triggered somehow. Options are event-driven (trigger immediately after each raw file or after schema profiling), or scheduled (run as a batch on a defined interval).

**Decision:**
Use an **EventBridge scheduled rule** to trigger the EMR promotion job on a defined schedule (e.g., every hour or daily, configurable per environment).

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| EventBridge Scheduled Rule (chosen) | Simple, predictable, EMR cluster spin-up cost amortized over a batch | Latency between raw landing and curated availability; files accumulate between runs |
| Event-driven (trigger on every S3 write) | Near-real-time curated data | Launching an EMR cluster per file is prohibitively expensive; not operationally viable |
| Event-driven after schema profiling (SNS → Step Functions → EMR) | Smarter batching, schema validation as gate | High complexity; Step Functions adds orchestration overhead; still needs a "collect enough files" strategy |
| Manual / on-demand | Maximum control | Not a production pattern; blocks pipeline automation |

**Consequences:**
- Data in the curated zone will be behind the raw zone by up to one schedule interval. This SLA must be communicated to downstream consumers.
- The promotion job processes all unpromoted files in the raw zone since the last successful run. It must track watermarks (e.g., using a DynamoDB bookmark or S3 prefix partitioned by date) to avoid reprocessing.
- `BREAKING` status schema records block promotion for that source. Other sources are not blocked — the job processes sources independently.

---

## ADR-005: Promotion Engine — EMR + PySpark vs. AWS Glue

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
The promotion step reads raw JSON/CSV from S3, validates against the schema registry, converts to Parquet, and writes to the curated zone. The two primary candidates for this workload on AWS are AWS Glue (managed Spark) and Amazon EMR (self-managed Spark cluster).

**Decision:**
Use **Amazon EMR with PySpark** for the promotion job, running as a transient cluster (spun up per job, terminated on completion).

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| Amazon EMR + PySpark (chosen) | Full control over Spark config and libraries, explicit DynamoDB read integration, no Glue-specific abstractions, good for learning raw Spark | Higher operational overhead; cluster startup time (~5-8 min); must manage cluster sizing |
| AWS Glue (managed Spark) | Fully managed, auto-scales, native Glue Catalog integration, DynamoDB connector available, lower ops burden | Glue DynamicFrame abstractions obscure Spark behavior; harder to customize; Glue-specific code is less portable |
| AWS Lambda (small files only) | Near-zero cold start for small payloads, no cluster overhead | Python Pandas/PyArrow on Lambda has memory limits; not viable for multi-GB files; not a general solution |

**Rationale for EMR over Glue:**
This project is explicitly a learning exercise. EMR forces direct engagement with Spark APIs, cluster configuration, and resource management. AWS Glue abstracts these away in ways that prevent learning the underlying mechanics. In a production environment owned by a mature data platform team, Glue would be the preferred default.

**Consequences:**
- The promotion job must handle atomic writes: use a staging prefix in S3, write Parquet there, then move to the curated prefix on job success. This prevents partial data from being visible to consumers.
- Cluster startup adds ~5-8 minutes to job latency. Acceptable given the scheduled (not real-time) trigger.
- EMR cluster role (EC2 instance profile) needs S3 read (raw), S3 write (curated + staging), DynamoDB read (schema registry), and Glue Catalog write permissions. IAM design is non-trivial.
- Transient clusters (spin-up → job → terminate) keep costs proportional to data volume. Persistent clusters would be cheaper at high frequency but wasteful at low frequency.

---

## ADR-006: Downstream Consumption Layer — Amazon Athena

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
After promotion, Parquet files in the curated zone need to be queryable by analysts and BI tools. Two primary options exist within the AWS ecosystem: Amazon Athena (serverless SQL over S3) and Amazon Redshift Spectrum (S3 queries through a Redshift cluster).

**Decision:**
Use **Amazon Athena** as the primary consumption layer, with curated zone tables registered in the AWS Glue Data Catalog.

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| Amazon Athena (chosen) | Serverless, no infrastructure to manage, pay-per-query, native S3 + Glue Catalog integration, accessible via JDBC for BI tools | Slower for complex joins vs. Redshift; per-query cost can grow at scale |
| Amazon Redshift Spectrum | Very fast for complex analytical queries, integrates with existing Redshift workloads | Requires a Redshift cluster (always-on cost); over-engineered if no existing Redshift investment |
| AWS Lake Formation | Adds fine-grained access control layer on top of Athena | Adds operational complexity; not needed for initial implementation |

**Consequences:**
- The promotion job must register new Parquet partitions in the Glue Catalog after each successful run (`MSCK REPAIR TABLE` or explicit partition registration via Glue API).
- Column naming in Parquet must be consistent with what Athena expects — lowercase, no special characters. Schema normalization should happen during promotion, not at query time.
- Query costs scale with data scanned. Parquet columnar format and S3 partitioning by `source` and `date` are the primary cost controls.

---

## ADR-007: Silent Schema Drift Detection and Alerting

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
The most dangerous failure mode is silent schema drift: a field is renamed or removed at the source, data continues to flow in, but downstream queries silently return nulls. The system must detect this and surface it to the responsible team before or during promotion.

**Decision:**
Implement a **two-stage detection and gating mechanism**:

1. **Detection (at profiling time):** The Lambda schema profiler diffs the incoming payload schema against the latest `ACTIVE` registry entry for that `source_id`. If any field from the prior version is absent in the new payload, write the new version with `status = BREAKING` and publish to SNS topic `schema-drift-alerts`.

2. **Gating (at promotion time):** The EMR promotion job checks the `status` field in the registry before processing each source. If any version in the promotion window has `status = BREAKING`, that source is skipped and a CloudWatch metric `PromotionBlockedByBreakingSchema` is emitted. Other sources are not blocked.

**Alerting Path:**
```
Lambda detects drift
    → writes BREAKING status to DynamoDB
    → publishes to SNS topic: schema-drift-alerts
        → CloudWatch Alarm (visible on ops dashboard)
        → Slack webhook (immediate team notification)
```

**Resolution Path:**
A data engineer reviews the `field_diff` in DynamoDB, determines whether the change is intentional (e.g., source team renamed a field), and either:
- Updates the registry entry to `status = ACTIVE` with an explanatory note (if the change is expected and downstream pipelines are updated), or
- Sets `status = DEPRECATED` on the old version and creates a migration plan for downstream consumers.

**Consequences:**
- No automatic healing. Schema drift always requires a human decision. This is intentional — automated resolution risks silently breaking downstream pipelines in the opposite direction.
- Downstream consumers must subscribe to `schema-drift-alerts` SNS if they want to be notified of changes that affect their pipelines.
- The `field_diff.renamed` heuristic will produce false positives (a removed field and a new field of the same type looks like a rename). Human review during resolution is the backstop for this.

---

## ADR-008: Error Handling — Dead-Letter Prefix and CloudWatch Metrics

**Date:** 2026-02-10
**Status:** Accepted

**Context:**
Files that cannot be profiled (corrupt JSON, invalid CSV, unrecognized format) must not be silently dropped. The system needs a path for failed records that preserves the original data and makes the failure observable.

**Decision:**
Use an **S3 dead-letter prefix** for files that fail profiling, combined with a CloudWatch custom metric.

- Failed files are moved to: `s3://raw-zone/dead-letter/source=<id>/date=<YYYY-MM-DD>/<original-filename>`
- The Lambda emits a custom CloudWatch metric: `Namespace=IngestionEngine, MetricName=ProfilingFailures, Dimensions=[source_id]`
- A CloudWatch Alarm on this metric notifies the `schema-drift-alerts` SNS topic if failures exceed a threshold.

**Alternatives Considered:**

| Option | Pros | Cons |
|---|---|---|
| S3 dead-letter prefix (chosen) | Simple, preserves original file, queryable for forensics, no additional services | Requires periodic cleanup process; not a true queue (no retry semantics) |
| SQS Dead Letter Queue | Built-in retry semantics, message TTL, standard AWS pattern | Files are not messages; SQS message size limit (256KB) makes storing raw file references preferable to content |
| Lambda Destinations (on-failure) | Native Lambda feature, no extra code | Still needs a storage target for the actual file; adds a layer without solving the storage problem |

**Consequences:**
- The dead-letter prefix must be monitored and periodically triaged. Files are not automatically retried.
- The original raw file is always preserved. A data engineer can re-trigger profiling manually after fixing the source of the malformation.
- Promotion job ignores the dead-letter prefix entirely — it only processes the normal raw prefix.
