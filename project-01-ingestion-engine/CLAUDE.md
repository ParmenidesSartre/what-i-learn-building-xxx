# Project 1: What I Learned Building a Schema-on-Read Ingestion Engine

---

## The Concept: Late Binding vs. Early Binding of Schema

Most data engineering courses teach you to define schemas upfront. But the fundamental tension in any data lake is *when* you bind structure to data. Early binding (schema-on-write) gives you safety but kills flexibility. Late binding (schema-on-read) gives you flexibility but pushes complexity downstream. The real lesson is understanding where in the pipeline you choose to pay the cost of structural validation, and why data lakes were designed around deferring that cost.

---

## The Project

Build an ingestion framework that accepts arbitrary JSON and CSV payloads into S3 (raw zone), performs structural profiling on arrival (via Lambda + EventBridge triggers), and writes a schema registry entry to a DynamoDB table — without ever rejecting data at the gate. Then build a "promotion" step that uses the registry to validate and convert raw data to Parquet in a curated zone using PySpark on EMR.

### Full Architecture (End-to-End)

```
[Source Systems]
       |
       v
[Ingestion Entry Point]          <-- API Gateway REST endpoint (see ADR-001)
       |
       v
[S3 Raw Zone]                    <-- unmodified, partitioned by source/date
       |
       | S3 Event Notification
       v
[Lambda: Schema Profiler]        <-- triggered on every PutObject (see ADR-002)
       |
       | writes
       v
[DynamoDB: Schema Registry]      <-- versioned, field-level schema store (see ADR-003)
       |
       | EventBridge scheduled rule (see ADR-004)
       v
[EMR + PySpark: Promotion Job]   <-- validates, converts to Parquet (see ADR-005)
       |
       v
[S3 Curated Zone]                <-- Parquet, partitioned, registered in Glue Catalog
       |
       v
[Athena / Redshift Spectrum]     <-- consumption layer for analysts and BI tools
```

---

## What Makes It Non-Trivial

The hard part isn't parsing JSON. It's designing the schema registry to handle schema evolution — what happens when the same source sends a payload tomorrow with two new fields and one renamed field?

### Schema Registry DynamoDB Design

| Attribute | Type | Purpose |
|---|---|---|
| `PK` | `source_id` (String) | Partition key — identifies the data source |
| `SK` | `version#<timestamp>` (String) | Sort key — enables ordered version history |
| `fields` | Map | Field name → inferred type |
| `field_diff` | Map | Added, removed, renamed fields vs. prior version |
| `status` | String | `ACTIVE`, `DEPRECATED`, `BREAKING` |
| `compatibility` | String | `BACKWARD`, `FORWARD`, `FULL`, `NONE` |
| `promoted_at` | String (ISO timestamp) | When this version was last promoted to curated |

**Versioning strategy:** Union-by-default with explicit breaking change flags (see ADR-003). New fields are additive and non-breaking. Renamed or removed fields are flagged as `BREAKING` and trigger an SNS alert before promotion proceeds.

---

## Failure Modes to Design For

### 1. Silent Schema Drift
A source system renames `transaction_date` to `txn_date` across a release. Your ingestion engine happily accepts both. Downstream, a pipeline reads `transaction_date`, gets nulls for all new records, and produces a report that looks correct but is missing 40% of data.

**Detection:** The Lambda profiler diffs the incoming schema against the latest `ACTIVE` registry entry for that source. If a field present in the prior version is absent in the new payload, it is flagged as a potential rename or removal.

**Surfacing:** An SNS topic (`schema-drift-alerts`) is published to. Subscribers include a CloudWatch alarm (for dashboards) and a Slack webhook (for the data engineering team). The DynamoDB record is written with `status = BREAKING` and the promotion job will refuse to proceed until the status is manually resolved or an override flag is set.

### 2. Promotion Failures / Malformed Payloads
Payloads that are not valid JSON or CSV at all (corrupted, wrong format) fail Lambda profiling. These are moved to an S3 dead-letter prefix (`s3://raw-zone/dead-letter/source=X/date=Y/`) and a CloudWatch metric is incremented. No silent data loss.

### 3. Partial Promotion
If the EMR job fails mid-run, no partial Parquet files are committed to the curated zone. The job uses a staging prefix and performs an atomic rename/move on success only.

---

## Gaps Acknowledged (Design Decisions Required)

The following areas require explicit decisions during implementation. Each must produce an entry in `ADR/decisions.md`:

1. **Ingestion entry point** — API Gateway vs. direct S3 upload vs. Kinesis Firehose
2. **Glue Catalog vs. custom DynamoDB registry** — why build custom instead of using the AWS standard
3. **Schema versioning strategy** — union vs. branch, backward/forward compatibility semantics
4. **EMR vs. AWS Glue for promotion** — operational trade-offs
5. **Promotion trigger mechanism** — scheduled vs. event-driven
6. **Downstream consumption layer** — Athena vs. Redshift Spectrum and query access patterns

---

## Claude Instructions

> **IMPORTANT — applies to all work on this project.**

When working on this project, you MUST follow these rules:

1. **Explain every decision.** For every architectural, structural, or technical choice made during implementation, state:
   - What the decision is
   - What alternatives were considered
   - Why this option was chosen
   - What trade-offs or consequences it introduces

2. **Write all decisions to the ADR file.** After making or refining any decision, append a new record to `ADR/decisions.md` using the standard ADR format:
   ```
   ## ADR-XXX: <Short Title>
   **Date:** YYYY-MM-DD
   **Status:** Proposed | Accepted | Deprecated | Superseded
   **Context:** <Why this decision was needed>
   **Decision:** <What was decided>
   **Alternatives Considered:** <What else was evaluated>
   **Consequences:** <What becomes easier, harder, or different as a result>
   ```

3. **Never skip the ADR step.** If you write code, configure infrastructure, or choose a library, that decision gets an ADR entry. No silent choices.

4. **Flag breaking changes.** If any decision contradicts or supersedes a prior ADR, update the prior ADR's status to `Superseded by ADR-XXX` and explain why.
