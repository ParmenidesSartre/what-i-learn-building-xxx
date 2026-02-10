# What I Learned Building [X]
**A Data Engineering Project Progression**

12 projects · concept-first · 2–3 weekends each
**Stack:** AWS (S3, EMR, Glue, Athena, Lambda, EventBridge, Redshift) · Python · PySpark · Terraform
**Domain:** Regulated Industry (Banking)

---

## Progression Map

Each project builds on knowledge from the previous ones. Projects 1–4 establish foundational plumbing (ingestion, lineage, idempotent processing, quality). Projects 5–6 go deeper into storage-layer internals. Projects 7–8 shift to governance and orchestration. Projects 9–12 cover platform maturity concerns that tie everything together.

| # | Project | Core Concept |
|---|---|---|
| 1 | [Ingestion Engine](project-01-ingestion-engine/) | Schema binding, raw/curated zones |
| 2 | [Lineage Tracker](project-02-lineage-tracker/) | Graph theory, audit trails |
| 3 | [Backfill System](project-03-backfill-system/) | Idempotency, event vs. processing time |
| 4 | [Data Quality Gate](project-04-data-quality-gate/) | Contracts, defensive engineering |
| 5 | [SCD Engine](project-05-scd-engine/) | Bi-temporality, historical truth |
| 6 | [Compaction Service](project-06-compaction-service/) | Storage physics, small file problem |
| 7 | [Access Control Layer](project-07-access-control-layer/) | ABAC, policy evaluation |
| 8 | [Event-Driven Orchestrator](project-08-event-driven-orchestrator/) | Dependency inversion, temporal decoupling |
| 9 | [Cost Attribution Engine](project-09-cost-attribution-engine/) | Platform economics, shared infra |
| 10 | [Pipeline Generator](project-10-pipeline-generator/) | Declarative design, DRY at scale |
| 11 | [Data Catalog Enrichment](project-11-data-catalog-enrichment/) | Metadata as control plane, classification theory |
| 12 | [SLO Observability](project-12-slo-observability/) | Error budgets, SRE for data |

---

## Project 1: What I Learned Building a Schema-on-Read Ingestion Engine

### The Concept: Late Binding vs. Early Binding of Schema

Most data engineering courses teach you to define schemas upfront. But the fundamental tension in any data lake is when you bind structure to data. Early binding (schema-on-write) gives you safety but kills flexibility. Late binding (schema-on-read) gives you flexibility but pushes complexity downstream. The real lesson is understanding where in the pipeline you choose to pay the cost of structural validation, and why data lakes were designed around deferring that cost.

### The Project

Build an ingestion framework that accepts arbitrary JSON and CSV payloads into S3 (raw zone), performs structural profiling on arrival (via Lambda + EventBridge triggers), and writes a schema registry entry to a DynamoDB table — without ever rejecting data at the gate. Then build a "promotion" step that uses the registry to validate and convert raw data to Parquet in a curated zone using PySpark on EMR.

### What Makes It Non-Trivial

The hard part isn't parsing JSON. It's designing the schema registry to handle schema evolution — what happens when the same source sends a payload tomorrow with two new fields and one renamed field? You need a versioning strategy. Do you union all versions? Do you branch? How do you communicate breaking changes downstream without coupling the ingestion layer to every consumer? Most people skip this and just overwrite schemas, which silently breaks pipelines weeks later.

### Failure Mode to Design For

Silent schema drift. A source system renames `transaction_date` to `txn_date` across a release. Your ingestion engine happily accepts both. Downstream, a pipeline reads `transaction_date`, gets nulls for all new records, and produces a report that looks correct but is missing 40% of data. Design for detecting and surfacing this.

---

## Project 2: What I Learned Building a File-Level Lineage Tracker

### The Concept: Data Lineage as a Graph Problem

Lineage isn't a logging exercise — it's a directed acyclic graph (DAG) problem. Every file in your lake is a node. Every transformation is an edge. The fundamental theory here is graph reachability: given a downstream report, can you walk backwards through the graph to identify every raw file that contributed to it? This is the theoretical foundation behind impact analysis, root cause tracing, and regulatory audit trails — all critical in banking.

### The Project

Instrument your ingestion engine from Project 1 to emit lineage events (source file → processed file → curated file) to a DynamoDB or S3-based lineage store. Each event records input files, output files, transformation ID, and timestamp. Build a CLI or small Python tool that, given any curated-zone file, traverses the graph backwards and returns the full lineage chain to raw sources.

### What Makes It Non-Trivial

Lineage gets complicated at fan-out and fan-in points. A single raw file might feed three curated tables (fan-out). A curated table might be built from 200 raw files across 15 batches (fan-in). Your graph traversal needs to handle both without exploding in complexity or producing misleading partial lineage. Most implementations only track "last hop" lineage, which is useless for audit.

### Failure Mode to Design For

Orphaned lineage. A pipeline reruns and overwrites a curated file, but the lineage store still points to the original raw inputs — not the reprocessed ones. Now your audit trail is wrong. Design for idempotent lineage updates that are tied to specific pipeline execution IDs, not just file paths.

---

## Project 3: What I Learned Building a Partition-Aware Backfill System

### The Concept: Idempotency and Temporal Partitioning

Idempotency means running an operation multiple times produces the same result as running it once. In data pipelines, this is deceptively hard because of temporal coupling — your pipeline's behavior depends on what time window it's processing. The deeper concept is understanding the difference between event time (when something happened) and processing time (when your pipeline ran), and why conflating them is the root cause of most backfill disasters.

### The Project

Build a PySpark job on EMR that processes transactional data partitioned by event date (year/month/day). Add a backfill mode where you can specify a date range, and the job cleanly reprocesses only those partitions — overwriting output partitions atomically using S3 path conventions and Glue catalog updates. Wire it with EventBridge + Lambda so that normal daily runs and backfill runs use the exact same code path, differing only in the date range parameter.

### What Makes It Non-Trivial

The hard part is partition atomicity on S3. S3 has no rename or transaction support. If your job writes new partition data but fails halfway, you have a half-written partition. If a consumer reads during the write, they get partial data. You need a write-audit-publish pattern: write to a staging prefix, validate, then "promote" by updating the Glue catalog pointer. Most people just overwrite in place and pray.

### Failure Mode to Design For

Late-arriving data. A banking transaction from January 15 arrives in your pipeline on February 3. Your daily pipeline for Feb 3 processes it, but it belongs in the Jan 15 partition. If your backfill system doesn't account for late arrivals, you either lose the record or double-count it. Design a late-arrival handling strategy with a configurable lookback window.

---

## Project 4: What I Learned Building a Data Quality Gate with Contract Testing

### The Concept: Contracts and Defensive Data Engineering

In software engineering, contract testing verifies that a producer and consumer agree on an interface. The same idea applies to data: a data contract is an explicit agreement about the shape, semantics, and quality of data at a boundary. The fundamental insight is that data quality is not something you bolt on after the fact — it's an architectural boundary between systems. Without contracts, every downstream consumer implicitly trusts every upstream producer, and trust without verification is the source of most data incidents.

### The Project

Build a YAML-defined data contract system. Each contract specifies: expected schema (column names, types), value constraints (non-null, range, regex, referential integrity), freshness SLA (data must arrive by X), and volume anomaly thresholds (row count within ±N% of trailing average). Implement a contract validator as a PySpark job that runs after ingestion (from Project 1) and before data enters the curated zone. Failed contracts write to a quarantine zone and emit alerts via SNS.

### What Makes It Non-Trivial

The hard part is calibrating thresholds without historical baselines. When you first deploy a quality gate, you don't know what "normal" looks like. Set thresholds too tight and you quarantine valid data, creating pipeline stalls. Set them too loose and you miss real issues. You need a bootstrapping phase where the system observes distributions before enforcing them. Most people hardcode thresholds from day one and spend weeks tuning false positives.

### Failure Mode to Design For

Referential integrity violations across partitions. A `customer_id` in your transactions table references a customer record that exists in yesterday's customer snapshot but was deleted in today's. Your quality gate passes each dataset independently, but the join downstream produces nulls. Design cross-dataset contract checks that validate referential integrity at the boundary, not just within a single table.

---

## Project 5: What I Learned Building a Slowly Changing Dimension Engine

### The Concept: Bi-Temporality and the Nature of Historical Truth

There are two timelines in any data system: valid time (when something was true in the real world) and transaction time (when the system recorded it). A Type 2 SCD only tracks valid time. Bi-temporal modeling tracks both. The fundamental insight is that in a regulated environment, you need to answer two different questions: "What was true on date X?" and "What did we believe was true on date X, given what we knew at time Y?" These are different questions with different answers, and conflating them is how banks fail audits.

### The Project

Build a PySpark job that implements a Type 2 SCD with bi-temporal columns: `valid_from`, `valid_to` (real-world validity), and `recorded_at`, `superseded_at` (system knowledge). Input is a daily full snapshot of customer dimension data landing in S3. The job merges it against the existing dimension table (stored as Parquet on S3, cataloged in Glue), detecting inserts, updates, and deletes. Deletes are soft-closed, not removed.

### What Makes It Non-Trivial

The merge logic with bi-temporal tracking is a state machine with more edge cases than people expect. What happens when a record is updated, then the update is retroactively corrected? You now have an amendment to a historical version — you need to close the old transaction-time record and open a new one with the corrected valid-time range, without disturbing the original valid-time history. Most SCD implementations handle happy-path inserts and updates but break on corrections, reinstatements, and same-day multi-updates.

### Failure Mode to Design For

Out-of-order snapshot delivery. Your source system delivers Tuesday's snapshot before Monday's due to a job scheduling issue. If your SCD engine processes them in arrival order, it will record incorrect state transitions — showing a change that "happened" on Tuesday, then "reverting" on Monday. Design for detection of out-of-order delivery and either resequencing or rejection.

---

## Project 6: What I Learned Building a Lake Storage Compaction Service

### The Concept: The Small File Problem and Storage Layer Physics

Object stores like S3 have fundamentally different performance characteristics than filesystems. Read throughput is governed by parallelism across objects, not sequential read speed. This means many small files (< 128MB) create massive overhead: each file requires a separate API call, and query engines like Athena/Spark spend more time opening connections than reading data. The concept here is storage-layer-aware pipeline design — understanding that how you physically organize bytes on disk (or object store) directly determines query performance, cost, and downstream pipeline speed.

### The Project

Build a compaction service triggered by EventBridge on a schedule. It scans partitions in your curated S3 zone, identifies partitions where the average file size is below a configurable threshold (e.g., 128MB), and runs a PySpark job on EMR to read and rewrite those partitions into optimally-sized files. It must do this without disrupting concurrent reads — use a staging-prefix-then-swap approach with Glue catalog updates. Track compaction history in DynamoDB (partition, before/after file count, before/after total size, timestamp).

### What Makes It Non-Trivial

The hard part is compaction under concurrent writes. If your ingestion pipeline writes new files to a partition while compaction is running, you risk either losing the new files (compaction reads old state, overwrites with compacted version that doesn't include new arrivals) or duplicating data. You need either a locking mechanism, a write-ahead log, or a design where compaction only targets partitions that are "sealed" (no longer receiving writes). This is the exact problem that table formats like Delta Lake and Iceberg solve — building it yourself teaches you why those formats exist.

### Failure Mode to Design For

Athena query cost explosion from un-compacted partitions. Athena charges per byte scanned, but small files cause it to scan inefficiently. A partition with 10,000 tiny files might cost 5x more to query than the same data compacted into 10 files, even though the total bytes are identical. Design the compaction service with a cost-tracking dimension: log estimated query cost before and after compaction so you can measure ROI.

---

## Project 7: What I Learned Building a Policy-Based Access Control Layer

### The Concept: Attribute-Based Access Control (ABAC) vs. Role-Based Access Control (RBAC)

RBAC assigns permissions to roles, and users inherit permissions through role membership. It works until you need fine-grained, context-dependent access — "analyst in the fraud team can see transaction amounts, but analyst in marketing cannot, and neither can see the customer's national ID." ABAC makes access decisions based on attributes of the user, the resource, and the environment. The fundamental concept is that access control is a policy evaluation problem, not a permissions lookup problem. In banking, this distinction is the difference between a working data platform and a compliance violation.

### The Project

Build a metadata-driven access control layer for your data lake. Define a policy schema in YAML: each policy specifies resource attributes (dataset, column, classification level), subject attributes (team, role, clearance level), and conditions (time-based, environment-based). Build a Python policy engine that, given a user context and a requested resource, evaluates all applicable policies and returns an allow/deny decision with an audit trail. Integrate it with a Glue catalog scanner that tags columns with classification levels (PII, confidential, internal, public) and a view generator that creates Athena views with column-level masking based on the caller's attributes.

### What Makes It Non-Trivial

The hard part is policy conflict resolution. When multiple policies apply to the same resource-user combination and they disagree, you need a deterministic resolution strategy. Deny-overrides? Most-specific-wins? Priority-based? Each strategy has different security implications. Most implementations either don't handle conflicts (last policy wins, nondeterministically) or use deny-overrides exclusively, which makes it nearly impossible to grant exceptions. You need to design an explicit conflict resolution strategy and make it auditable.

### Failure Mode to Design For

Column-level policy bypass through derived columns. A policy denies access to `national_id`. A user creates a derived table that includes `hashed_national_id = sha256(national_id)`. The policy engine doesn't see the derivation, so access is granted. But a rainbow table attack can reverse the hash. Design for policy propagation through lineage — use the lineage graph from Project 2 to detect when a restricted column feeds into a derived column and propagate the restriction.

---

## Project 8: What I Learned Building an Event-Driven Pipeline Orchestrator

### The Concept: Event-Driven vs. Schedule-Driven Orchestration and the Dependency Inversion Principle

Most pipelines are scheduled: "run Job B at 3:00 AM because Job A usually finishes by 2:45 AM." This creates temporal coupling — Job B doesn't actually depend on the clock, it depends on Job A's output existing. The fundamental insight is the dependency inversion principle applied to data pipelines: instead of jobs knowing about each other's schedules, each job should declare what it produces and what it requires. An orchestrator resolves dependencies at runtime based on actual data availability, not assumed timing. This is the difference between a fragile pipeline chain and a resilient data platform.

### The Project

Build an event-driven orchestrator using EventBridge, Lambda, DynamoDB, and Step Functions. Each pipeline publishes a "data ready" event when it completes (including dataset name, partition, schema version, row count). Downstream pipelines register their dependencies as rules in DynamoDB: "I need dataset A partition X and dataset B partition X both to exist before I run." A Lambda evaluator listens for all "data ready" events, checks the dependency table, and triggers downstream pipelines only when all dependencies are satisfied. Include a dead-letter mechanism for events that never satisfy their dependencies within a timeout.

### What Makes It Non-Trivial

The hard part is partial dependency satisfaction with timeout semantics. If Pipeline C depends on outputs from Pipeline A and Pipeline B, and Pipeline A completes but Pipeline B fails, Pipeline C is stuck. How long do you wait? Do you retry Pipeline B? Do you run Pipeline C with partial data? The design must handle dependency timeout, partial execution policies, and cascading failure isolation. Most event-driven systems only handle the happy path where all producers deliver on time.

### Failure Mode to Design For

Event replay and duplicate triggering. EventBridge guarantees at-least-once delivery. If a "data ready" event is delivered twice, your orchestrator might trigger the downstream pipeline twice. If that pipeline isn't idempotent (see Project 3), you get duplicate data. Design for event deduplication using idempotency keys in DynamoDB with conditional writes.

---

## Project 9: What I Learned Building a Cost Attribution Engine for a Shared Data Platform

### The Concept: Chargeback, Showback, and the Economics of Shared Infrastructure

When multiple teams share a data platform, the tragedy of the commons applies: no individual team bears the cost of their usage, so consumption grows unchecked. The fundamental economic concept is cost attribution — mapping shared infrastructure spend to the teams, pipelines, and datasets that caused it. Chargeback (billing teams directly) changes behavior. Showback (showing teams their costs without billing) changes awareness. The architectural challenge is that cloud costs are reported at the resource level (this EMR cluster cost $X), not at the logical level (this pipeline for team Y cost $X).

### The Project

Build a cost attribution engine that ingests AWS Cost and Usage Reports (CUR) from S3, enriches them with pipeline metadata (which pipeline ran on which EMR cluster, which dataset lives in which S3 prefix, which Athena queries came from which team), and produces a per-team, per-pipeline, per-dataset cost breakdown. Use tagging conventions on AWS resources, combine with Glue catalog metadata and CloudTrail logs for Athena query attribution. Output to a cost ledger in S3 (Parquet, partitioned by month and team).

### What Makes It Non-Trivial

The hard part is shared resource disaggregation. An EMR cluster runs 5 Spark jobs for 3 different teams in a single hour. The CUR reports one line item for that cluster-hour. How do you split it? By wall-clock time per job? By CPU-seconds? By data volume processed? Each method gives different numbers, and teams will dispute any allocation they perceive as unfair. You need a transparent, defensible allocation methodology and the data to back it up — which means your pipelines need to emit resource consumption metrics (from Spark's REST API or event logs), not just completion events.

### Failure Mode to Design For

Untagged resources creating a cost black hole. If 30% of your EMR clusters aren't tagged with a team or pipeline, 30% of your cost is unattributable. It gets dumped into "shared/unallocated," which nobody owns and nobody optimizes. Design a tagging compliance checker that runs before cost attribution and flags untagged resources with escalating alerts.

---

## Project 10: What I Learned Building a Metadata-Driven Pipeline Generator

### The Concept: Declarative vs. Imperative Pipeline Design and the DRY Principle

When you have 50 ingestion pipelines that all do roughly the same thing — land a file, validate schema, convert to Parquet, register in catalog — writing each one imperatively is a maintenance nightmare. The fundamental concept is declarative pipeline specification: instead of writing code that says how to ingest, you write configuration that says what to ingest, and a generator produces the executable pipeline. This is the DRY (Don't Repeat Yourself) principle applied at the architecture level. It's also the core idea behind every internal data platform tool at companies like Netflix, Airbnb, and Uber.

### The Project

Build a pipeline generator that reads YAML pipeline definitions (source type, source location, target zone, schema contract reference, partitioning strategy, SLA, owner) and generates: the PySpark job code (from Jinja2 templates), the Terraform resource definitions for Lambda triggers and EventBridge rules, the Glue catalog table definition, and the data contract YAML (referencing Project 4). A single CLI command — `generate --config pipeline.yaml` — should produce all deployment artifacts. A second command — `deploy --config pipeline.yaml` — should apply them via Terraform.

### What Makes It Non-Trivial

The hard part is balancing generalization with escape hatches. The moment you make a generator, someone needs a pipeline that doesn't fit the template — a custom transformation, a non-standard partitioning scheme, a source with authentication quirks. If your generator is too rigid, people bypass it and write custom pipelines (defeating the purpose). If it's too flexible, it becomes a poorly designed programming language. Design for a plugin/hook system where 80% of pipelines are pure config and 20% can inject custom code at well-defined extension points.

### Failure Mode to Design For

Template drift. You fix a bug in the PySpark template, but the 30 pipelines already generated from the old template are deployed and running the old code. Now you have two versions in production. Design for template versioning and a mechanism to detect which deployed pipelines are running outdated generated code, with a regeneration and redeployment path.

---

## Project 11: What I Learned Building a Data Catalog with Automated Classification

### The Concept: Metadata as a First-Class Citizen and Information Theory in Classification

Most teams treat metadata as an afterthought — column names and types in a catalog, maybe some descriptions if you're lucky. The fundamental concept is that metadata is the control plane of your data platform. Lineage (Project 2), access policies (Project 7), quality contracts (Project 4), cost attribution (Project 9), and pipeline generation (Project 10) all depend on rich, accurate metadata. Automated classification applies basic information theory — examining the statistical properties of column values (entropy, cardinality, pattern distribution) to infer semantic types (email, phone, national ID, currency) without human labeling.

### The Project

Build a catalog enrichment service that scans datasets registered in your Glue catalog. For each column, it samples values and computes: cardinality ratio (unique values / total rows), entropy (Shannon entropy of the value distribution), pattern frequency (regex matching for known formats like emails, IBANs, phone numbers, Malaysian IC numbers), and null ratio. Based on these signals, it classifies columns into semantic types and sensitivity levels, writing the classification back to Glue catalog as table/column properties. This feeds directly into the access control layer from Project 7.

### What Makes It Non-Trivial

The hard part is disambiguation. A column of 12-digit numbers could be a phone number, an account number, or an IC number. Pattern matching alone isn't sufficient — you need contextual signals like column name, co-occurring columns (a column next to `phone_country_code` is probably a phone number), and table-level context. Building a classification system that handles ambiguity honestly — assigning confidence scores rather than hard labels — is the real challenge. Most implementations use regex-only matching and produce embarrassing false positives.

### Failure Mode to Design For

Classification staleness. You classify all columns on Monday. On Wednesday, a new column appears (from schema evolution in Project 1), or a column's value distribution changes enough that its classification should change (a "notes" field starts containing IC numbers pasted by careless operators). Design for incremental reclassification triggered by schema change events and value distribution drift detection.

---

## Project 12: What I Learned Building a Platform Health Dashboard with SLO-Based Observability

### The Concept: SLIs, SLOs, Error Budgets, and the Theory of Observability

Observability isn't monitoring with more dashboards. The fundamental theory (from Google's SRE framework) distinguishes between Service Level Indicators (measurable properties: latency, freshness, completeness), Service Level Objectives (target values: "data freshness < 2 hours for 99.5% of datasets"), and Error Budgets (how much failure is acceptable before you stop shipping features and fix reliability). Applied to data platforms, this means defining what "healthy" means quantitatively for your data lake — not "are the pipelines green" but "are we meeting our commitments to data consumers, and how much room do we have before we breach them?"

### The Project

Build an observability layer that collects SLIs from all previous projects: data freshness (time between source event and curated zone availability, from Project 1/3), data quality pass rate (from Project 4), pipeline success rate and latency (from Project 8), cost per dataset trend (from Project 9), and catalog completeness (percentage of columns with classification, from Project 11). Define SLOs for each SLI in YAML. Build a Python aggregator (Lambda-based, scheduled) that computes current SLI values, compares against SLOs, maintains a rolling error budget, and writes results to S3. The deliverable dashboard (simple HTML served from S3, or Athena-queryable tables) is secondary — the real deliverable is the SLO framework and error budget calculation engine.

### What Makes It Non-Trivial

The hard part is choosing the right SLIs and setting SLOs that actually matter. It's tempting to measure everything and set aggressive targets. But an SLO of 99.9% freshness for a dataset that consumers check weekly is wasted effort, while 99% completeness for a regulatory report might be dangerously loose. You need to work backwards from consumer expectations and regulatory requirements to set SLOs — which means your observability system needs to understand the importance of datasets, not just their technical properties. This requires integrating with the metadata catalog from Project 11 and the data contracts from Project 4.

### Failure Mode to Design For

Alert fatigue from SLO breaches that don't matter. If every minor SLO breach triggers a PagerDuty alert, your team learns to ignore them. Design a tiered alerting system based on error budget burn rate — a slow burn sends a weekly digest, a fast burn sends an immediate alert, and an exhausted error budget triggers an incident. The math behind burn rate alerting (multi-window, multi-burn-rate) is where the real learning is.
