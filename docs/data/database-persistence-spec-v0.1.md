# Trackfy — Database & Persistence Specification v0.1

**Status:** Draft  
**Version:** 0.1  
**Scope:** Transactional persistence, canonical event durability, tenant isolation, idempotency, lifecycle projections, analytics handoff and persistence operational rules.

This document translates the Domain Model, Event/Tracking Contracts, Multi-tenancy/RBAC and Architecture into a persistence contract. It is intentionally database-design level, not a final SQL migration set. Physical choices may evolve only through an explicit architecture change.

## 1. Persistence goals

Trackfy persistence must guarantee:

1. tenant-owned data has an unambiguous authorization scope;
2. canonical event evidence remains durable and immutable;
3. derived state can be rebuilt from canonical facts;
4. retries do not create duplicate business outcomes;
5. operational reads do not depend on scanning raw event history unnecessarily;
6. high-volume analytics does not starve transactional workloads;
7. schema changes are migration-controlled and reversible where practical;
8. deletion and retention can be enforced without violating auditability requirements;
9. background workers can safely resume after failure;
10. every persistent projection has a defined source of truth.

## 2. Chosen persistence direction

The current stack is:

```text
PostgreSQL 18.x
  ↓
transactional / control-plane state

Redis
  ↓
cache + rate limits + transient async coordination

ClickHouse
  ↓
analytical/event-query workloads

S3-compatible object storage
  ↓
exports / large artifacts / optional archival evidence
```

PostgreSQL is the primary transactional database. It is not assumed to be the only physical representation of events.

ClickHouse is an analytical projection target, not an independent source of truth.

Redis is never authoritative for business facts.

## 3. Source-of-truth hierarchy

```text
External provider assertion
        ↓
Raw evidence
        ↓
Canonical event / business fact
        ↓
Derived projection
        ↓
Cache / dashboard representation
```

For tracking, accepted canonical events are the durable internal source of truth.

For provider-owned business facts such as payment status, the external provider may remain authoritative for the fact itself. Trackfy persists the assertion it received and normalized.

No cache, dashboard row, aggregate or current-state projection may supersede the canonical evidence.

## 4. PostgreSQL responsibility boundary

PostgreSQL owns transactional entities and state requiring relational integrity.

Initial conceptual domains:

```text
Identity
  users

Tenancy
  organizations
  organization_memberships
  workspaces

Tracking configuration
  sites
  tracking_configurations
  domains / origins

Integrations
  integrations
  integration_credentials_metadata
  webhook_endpoints
  api_keys_metadata

Commerce
  orders
  order_status_history
  refunds
  fees / commissions / costs

Configuration
  attribution_policies
  routing_rules
  dashboards
  alerts / rules

Reliability
  idempotency_records
  ingestion_records
  processing_records
  delivery_records
```

Exact table names are provisional. The ownership and invariants are the important contract.

## 5. Tenant boundary

Every tenant-owned record must carry or inherit an authorization path that resolves to exactly one Organization.

Preferred hierarchy:

```text
Organization
 └── Workspace
      ├── Site
      ├── Integration
      ├── API credential scope
      └── operational data
```

A resource may be Organization-scoped or Workspace-scoped, but its scope must be explicit.

No query may rely on the caller merely supplying an `organization_id` or `workspace_id`. The application must derive and validate authorization from authenticated membership and resource ownership.

## 6. Identifier strategy

Application identifiers should be opaque, globally collision-resistant and externally safe.

Conceptual prefixes:

```text
usr_   user
org_   organization
ws_    workspace
site_  site
int_   integration
vis_   visitor
ses_   session
clk_   click
evt_   event
ord_   order
ref_   refund
```

The database may use UUID/ULID-like physical values, but API identifiers should not expose implementation-specific sequential IDs where avoidable.

Internal foreign keys may use compact native database types if the mapping remains safe and stable.

## 7. Time and audit fields

Persistent operational records should distinguish:

```text
created_at
updated_at
occurred_at       (when domain-appropriate)
received_at       (ingestion)
processed_at      (processing)
```

`created_at` and `updated_at` are storage metadata. They must not substitute for domain occurrence time.

Lifecycle transitions require their own effective/source timestamp where supplied by an authoritative source.

## 8. Canonical event persistence

Canonical events are append-only facts.

Conceptual record:

```text
event_id
organization_id
workspace_id
site_id

name
version
occurred_at
received_at

source_type
source_provider
integration_id

visitor_id
session_id
click_id

acquisition snapshot
context snapshot
payload
provenance

created_at
```

Rules:

1. `event_id` is unique within the canonical event store.
2. Accepted canonical events are never updated in place to alter historical meaning.
3. Enrichment belongs in derived fields/projections or immutable enrichment records rather than silently overwriting evidence.
4. Provider/raw references remain resolvable under retention policy.
5. The original contract version remains attached to the event.

## 9. Raw evidence persistence

Raw evidence may be stored separately from the canonical relational model when payloads are large or provider-specific.

Conceptual metadata:

```text
raw_record_id
organization_id
workspace_id
integration_id
source_type
provider
received_at
external_event_id
payload_hash
object_reference
schema_version
```

Raw payload storage must be access-controlled and retention-governed.

The raw record should be content-addressable or otherwise integrity-verifiable where practical.

## 10. Event immutability strategy

Database permissions and repository-level data access must prevent ordinary application paths from modifying canonical evidence.

Corrections occur by adding new facts or derived correction records, for example:

```text
original event
    ↓
correction / reversal event
    ↓
recomputed projection
```

A mutable administrative edit must never erase the historical value that downstream systems may have relied upon.

## 11. Idempotency records

Mutating API operations that can be retried require an idempotency record.

Conceptual fields:

```text
scope_key
idempotency_key
request_fingerprint
response_status
response_body/reference
resource_id
created_at
expires_at
```

Required constraints:

```text
UNIQUE(scope_key, idempotency_key)
```

A replay with the same key and same fingerprint returns the previously established logical outcome.

A replay with the same key and different fingerprint fails deterministically.

Idempotency records are operational safety state. They are not business facts.

## 12. Ingestion records

The ingestion boundary should retain enough metadata to diagnose delivery without confusing transport with canonical facts.

Conceptual fields:

```text
ingestion_id
request_id
organization_id
workspace_id
site_id / integration_id
source
received_at
payload_hash
event_id if present
http/status outcome
validation outcome
```

This record supports operational tracing and duplicate diagnosis.

## 13. Processing records

Asynchronous processing must be resumable and observable.

Conceptual fields:

```text
processing_id
event_id
stage
attempt
status
error_code
started_at
finished_at
next_retry_at
worker_version
```

The same event may have multiple processing attempts, but attempts must not create duplicate business effects.

## 14. Orders and lifecycle persistence

Orders are transactional business entities, not just event payloads.

Conceptual Order state:

```text
order_id
organization_id
workspace_id
site_id
external_order_id
provider/integration
customer reference
currency
gross amount
current canonical status
created_at
updated_at
```

Lifecycle history is separate:

```text
order_status_history
  ├── pending
  ├── approved
  ├── rejected
  ├── cancelled
  ├── refunded
  ├── partially_refunded
  └── chargeback
```

The current status is a projection over lifecycle facts; it is not the only representation of history.

## 15. Financial adjustments

Refunds, fees, commissions and product costs must remain distinguishable.

A conceptual adjustment record should preserve:

```text
adjustment_id
order_id
organization_id
workspace_id
type
amount
currency
occurred_at
source
external_reference
```

This allows reporting to distinguish gross revenue, refunds, fees, commissions and other costs without embedding accounting semantics into a single mutable Order total.

Detailed financial ledger semantics are a future dedicated specification.

## 16. Visitor / Session / Click persistence

These records support operational queries and correlation.

### Visitor
A measurement identity scoped to a Site/Workspace according to the tracking policy.

### Session
A bounded interaction projection derived from canonical events and sessionization rules.

### Click
An acquisition identity retaining Trackfy `click_id` plus provider identifiers and acquisition evidence.

The persistence layer must allow these projections to be recomputed when sessionization or correlation policy changes.

## 17. Campaign and acquisition metadata

Campaign/source dimensions used for reporting should preserve both:

```text
raw received acquisition evidence

and

derived normalized reporting dimensions
```

This prevents reporting normalization from destroying original UTM or click-ID evidence.

## 18. PostgreSQL constraints

The initial schema design should enforce at the database layer whenever possible:

- primary keys;
- foreign keys;
- unique identifiers;
- scoped uniqueness for external IDs;
- non-null invariants for fields required by the domain;
- valid enum/state constraints where stable;
- positive/non-negative checks where semantically valid;
- currency presence alongside money amounts;
- tenant consistency across related rows.

Application validation remains necessary; database constraints are a second line of defense, not a replacement for authorization.

## 19. Tenant consistency constraints

Where two records are related and both carry tenant scope, their Organization/Workspace ownership must agree.

Unsafe pattern:

```text
order.workspace_id = A
order.customer.workspace_id = B
```

The persistence model should prevent such cross-scope relationships through composite foreign keys, database constraints or transaction-level application invariants.

## 20. Indexing principles

Indexes should follow verified access patterns rather than speculative completeness.

High-priority access patterns include:

```text
Organization → Workspaces
Workspace → Sites
Workspace → Orders
Site → recent events
Visitor → recent sessions
Session → timeline events
Click → downstream events/orders
Order → lifecycle history
Event → event_id
Event → tenant + occurred_at
Processing → pending/retryable work
Delivery → event + destination + status
```

Tenant scope should be part of important composite indexes to prevent broad scans across unrelated customers.

## 21. Event indexing and partitioning

Canonical event volume may eventually exceed what is comfortable in ordinary PostgreSQL tables.

The logical contract therefore permits partitioned PostgreSQL event storage or a separate analytical event store.

Partitioning key candidates include time plus tenant/site locality.

The partitioning strategy must preserve:

- stable `event_id` lookup;
- tenant filtering;
- replayability;
- retention management;
- predictable pruning.

Partitioning is an implementation decision to validate with real volume.

## 22. PostgreSQL ↔ ClickHouse boundary

The preferred flow is:

```text
Canonical event
       │
       ├── PostgreSQL operational persistence
       │
       └── analytics projection → ClickHouse
```

ClickHouse may hold:

- event history optimized for analytics;
- denormalized analytical dimensions;
- time-series aggregates;
- funnel projections;
- campaign performance projections;
- tracking health metrics.

ClickHouse must not become the authority for tenant configuration, RBAC or mutable transactional lifecycle state.

## 23. Analytics projection semantics

Analytics projections are rebuildable.

A projection pipeline should retain enough metadata to determine:

```text
source event / source revision
projection version
projection timestamp
```

When a projection is stale or corrupted:

```text
canonical facts
      ↓
rebuild job
      ↓
new projection
```

The UI should tolerate brief eventual-consistency windows where the product contract permits them.

## 24. Read models

The Dashboard and Event Debugger should use purpose-built read models for expensive queries.

Examples:

```text
daily_campaign_metrics
order_dashboard_summary
event_trace_summary
tracking_health_snapshot
integration_delivery_summary
```

These are derived and replaceable. They must not be edited manually as authoritative business records.

## 25. Transaction boundaries

A database transaction should cover a logically atomic business effect.

Examples:

```text
accept canonical event metadata
 + record deduplication decision
```

or:

```text
apply one Order lifecycle transition
 + persist transition evidence
 + update current projection
```

Large analytical work must not remain inside the same OLTP transaction.

## 26. Outbox / durable handoff

When an accepted transaction must trigger asynchronous side effects, a durable handoff pattern should be used rather than relying on in-memory callbacks.

Conceptual flow:

```text
DB transaction
  ├── canonical/transactional write
  └── outbox record
          ↓
      dispatcher
          ↓
        queue
          ↓
       worker
```

This avoids the class of failure where the database commit succeeds but event publication is lost.

The exact outbox implementation is part of the persistence engineering design.

## 27. Redis boundary

Redis may contain:

- short-lived cache entries;
- rate-limit counters;
- transient locks/leases;
- retry scheduling metadata;
- queue/stream state where selected.

Redis must not be the sole persistence location for:

- canonical events;
- Orders;
- payment lifecycle;
- attribution results;
- tenant membership;
- API credentials metadata.

A Redis loss must be recoverable without loss of canonical business truth.

## 28. Object storage boundary

S3-compatible storage is appropriate for large or immutable artifacts such as:

- exports;
- large raw provider payloads;
- diagnostic bundles;
- backups/snapshots where applicable.

Object references stored in PostgreSQL must include tenant scope and integrity metadata when required.

## 29. Retention and deletion

Retention policy must distinguish:

```text
canonical events
raw evidence
operational projections
analytics projections
idempotency records
logs / diagnostics
exports
```

Deletion of derived projections must be safe because they can be rebuilt where canonical evidence remains.

Deletion of canonical/raw evidence requires an explicit retention/privacy policy and may affect replayability.

## 30. Migration discipline

All PostgreSQL schema changes must be migration-controlled.

Rules:

1. never mutate production schema manually as the normal workflow;
2. migrations are ordered and committed with the code that requires them;
3. destructive migrations require explicit review and a rollback/forward-recovery plan;
4. data migrations are separated when operationally necessary;
5. compatibility periods are used for breaking application/schema changes;
6. migration tests run in CI.

The final migration tooling is implementation-specific but must support reproducible environments.

## 31. Backup and recovery requirements

Production persistence must have:

- automated backups;
- restore verification;
- documented recovery objectives;
- independent protection from application credentials where practical;
- recovery drills before production readiness.

Backup availability is not equivalent to tested recoverability.

## 32. Security boundary

Database access must be role-separated.

Application roles should receive only the privileges needed for their function.

Direct public database access is prohibited.

Credential secrets must not be persisted as plaintext when a managed secret reference or encrypted representation is available.

Raw event access and exported data require tenant- and role-aware authorization.

## 33. Concurrency and locking

The persistence model must handle concurrent operations such as:

- two retries of one idempotent request;
- two lifecycle updates for the same Order;
- simultaneous membership changes;
- duplicate webhook delivery;
- concurrent processing workers.

Locks or unique constraints should protect narrowly defined invariants rather than serializing whole tenants unnecessarily.

## 34. Delete semantics

Deletion behavior must be explicit for each relation.

Do not apply blanket cascading deletion to all tenant data.

For example:

```text
delete Workspace
  ≠ automatically destroy canonical events immediately
```

Retention, legal/privacy requirements and export obligations may require staged deletion, anonymization or tombstoning.

## 35. Persistence failure behavior

Required properties:

```text
Postgres unavailable
  → ingestion must fail safely or enter an explicitly bounded buffering mode

Redis unavailable
  → authoritative data remains safe; non-critical cache/queue degradation is observable

ClickHouse unavailable
  → canonical/operational tracking remains available where product SLA permits

Object storage unavailable
  → large-artifact workflows fail explicitly without corrupting transactional facts
```

No downstream analytics or integration outage may silently mark a canonical event as lost.

## 36. Acceptance criteria

### AC-01 — Tenant isolation
A query executed with valid membership for Workspace A cannot return resources owned by Workspace B.

### AC-02 — Scoped uniqueness
The same external identifier may exist under different providers/integrations but cannot collide inside the same required scope.

### AC-03 — Event immutability
An accepted canonical event cannot be changed through ordinary application write paths.

### AC-04 — Idempotent retry
A repeated mutating request with identical idempotency key and fingerprint produces one logical business outcome.

### AC-05 — Conflicting retry
An idempotency key reused with materially different request content is rejected.

### AC-06 — Lifecycle history
An Order transition creates durable history and updates its current-state projection without deleting prior state.

### AC-07 — Projection rebuild
A derived read model can be deleted and rebuilt from canonical facts with equivalent output for the same projection version.

### AC-08 — Analytics isolation
A slow analytical query cannot hold the transaction required to acknowledge a tracking event indefinitely.

### AC-09 — Durable handoff
A committed transactional fact that requires async processing leaves enough durable state for the worker to resume after process failure.

### AC-10 — Recovery
A fresh environment can be brought to the expected schema through migrations alone, without manual database edits.

## 37. Required schema-test matrix

Before implementation is considered complete, automated tests must cover at least:

```text
tenant boundary
scoped FK consistency
unique event_id
idempotency replay
idempotency conflict
order lifecycle transition
refund adjustment
immutable event write rejection
outbox persistence
projection rebuild
migration reproducibility
```

## 38. Deferred persistence decisions

The following remain explicitly open until measured or specified further:

- exact PostgreSQL schema/table names;
- UUID versus ULID physical representation;
- exact event partitioning strategy;
- whether canonical events live first in PostgreSQL, ClickHouse, or both physically;
- queue implementation;
- outbox transport implementation;
- exact ClickHouse table design;
- retention periods;
- backup provider and RPO/RTO targets;
- encryption implementation;
- secret-management provider;
- final migration tool;
- financial ledger depth.

## 39. Next step

The next implementation artifact should be **Database Contract Tests v0.1** followed by the first PostgreSQL migration slice. The tests must make the tenant, identity, event, idempotency and lifecycle invariants executable before the application repositories are implemented.
