# Trackfy — Architecture v0.1

**Status:** Draft  
**Version:** 0.1  
**Scope:** Logical system architecture for tracking collection, canonical event processing, attribution, analytics, integrations, multi-tenant control and dashboard delivery.

This document translates the Product Spec, Domain Model, Event/Tracking Contracts and Attribution Model into architectural boundaries. It is not a deployment manifest, database schema or final technology selection.

## 1. Architectural objective

Trackfy must support high-volume event collection without allowing tracking concerns, tenant administration, analytics queries, third-party integrations and dashboard rendering to become one coupled system.

The architecture therefore separates five major planes/capabilities:

```text
                    Trackfy
                       │
       ┌───────────────┼────────────────┐
       │               │                │
  Control Plane     Data Plane       Analytics
       │               │                │
       │               │                │
       ├───────────────┼────────────────┤
       │               │                │
       └──────── Integrations ──────────┘
                       │
                    Dashboard
```

The boundaries are logical first. A physical deployment may colocate components where scale and operational simplicity permit.

## 2. Core architectural principles

1. Canonical events are durable evidence; derived state is recomputable.
2. Tracking ingestion is isolated from interactive dashboard workloads.
3. Tenant authorization is enforced at every control/data boundary.
4. External providers are adapters, not sources of Trackfy's canonical domain model.
5. Attribution and analytics consume canonical facts; they do not mutate them.
6. Delivery failures must not invalidate canonical business facts.
7. The simplest architecture satisfying observed requirements is preferred.
8. Every asynchronous boundary must be observable and replayable where appropriate.
9. Contracts precede implementation and integration-specific schemas remain outside the canonical model.
10. A read-optimized analytics representation may diverge physically from the transactional control model without becoming a second source of truth.

## 3. Logical component map

```text
                     ┌──────────────────────┐
                     │      Customers       │
                     │ Websites / Apps      │
                     └──────────┬───────────┘
                                │
                  Browser SDK / GTM / S2S
                                │
                                ▼
                    ┌────────────────────┐
                    │ Tracking Edge /    │
                    │ Ingestion Gateway  │
                    └─────────┬──────────┘
                              │
                     auth / limits / validate
                              │
                              ▼
                    ┌────────────────────┐
                    │ Raw Evidence       │
                    │ Boundary           │
                    └─────────┬──────────┘
                              │
                              ▼
                    ┌────────────────────┐
                    │ Canonical Event    │
                    │ Pipeline           │
                    └─────────┬──────────┘
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    Identity /            Enrichment /       Sessionization
    Correlation           Normalize
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                    ┌────────────────────┐
                    │ Derived Processing │
                    └─────────┬──────────┘
                              │
                ┌─────────────┼─────────────┐
                ▼             ▼             ▼
          Attribution     Lifecycle      Analytics
                │             │             │
                └─────────────┼─────────────┘
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
         Integration Router            Event Debugger
                │                           │
      ┌─────────┼─────────┐                 │
      ▼         ▼         ▼                 ▼
    Meta     Google    TikTok          Tracking Health
    /CAPI     /Ads      /Events
      │         │         │
      └─────────┴─────────┘
                │
                ▼
        External Webhooks / CRM

              ┌────────────────────────────┐
              │        Control Plane        │
              │ organizations / workspaces │
              │ members / RBAC / sites     │
              │ integrations / credentials │
              │ configuration / reporting  │
              └─────────────┬──────────────┘
                            │
                            ▼
                       Dashboard/API
```

## 4. Control Plane

The Control Plane manages configuration and tenant-owned operational state rather than high-volume raw event ingestion.

Responsibilities:

- User and Organization management;
- Workspace management;
- Members and RBAC;
- Sites and tracking configuration;
- integration configuration;
- API key lifecycle;
- webhook configuration;
- campaign/account metadata;
- attribution configuration;
- reporting/dashboard configuration;
- alert/rule configuration;
- billing/product entitlements when introduced.

The Control Plane should expose a stable application API for the dashboard and administrative clients.

The Control Plane must not become the hot path for every tracking event.

## 5. Data Plane

The Data Plane handles event collection and processing.

Responsibilities:

- browser ingestion;
- GTM Web ingestion;
- GTM Server ingestion;
- S2S API ingestion;
- provider webhooks;
- offline/backfill ingestion;
- transport validation;
- event authentication;
- tenant/site resolution;
- deduplication/idempotency;
- raw evidence persistence;
- canonical event creation;
- asynchronous processing;
- event correlation;
- downstream delivery.

The Data Plane must be independently scalable from dashboard traffic.

## 6. Tracking Edge / Ingestion Gateway

The first server boundary should be optimized for high-throughput, low-latency ingestion.

Responsibilities:

1. Resolve the target Site/integration.
2. Authenticate where required.
3. Apply request limits and abuse controls.
4. Reject malformed requests early.
5. Assign authoritative `received_at`.
6. Preserve raw evidence.
7. Establish an ingestion identity/idempotency decision.
8. Hand accepted evidence to canonical event processing.

The gateway must not perform expensive analytics queries synchronously.

## 7. Raw Evidence Layer

Raw provider/browser/server assertions are retained as traceable evidence before normalization.

The raw layer exists to support:

- debugging;
- replay;
- adapter evolution;
- forensic investigation;
- correction of derived projections;
- provenance.

Retention and privacy policy are separate specifications.

Raw evidence is not exposed universally. Access is tenant- and role-controlled.

## 8. Canonical Event Layer

The Canonical Event Layer is the internal contract boundary defined by Event/Tracking Contracts.

Each accepted event has:

- immutable event identity;
- event type/version;
- occurrence/receipt timestamps;
- tenant context;
- identity context;
- acquisition evidence;
- business payload;
- provenance.

Downstream components consume this contract instead of coupling directly to provider payloads.

## 9. Processing pipeline

Processing should be logically staged:

```text
Ingest
  ↓
Validate
  ↓
Persist evidence
  ↓
Canonicalize
  ↓
Correlate
  ↓
Enrich
  ↓
Project lifecycle
  ↓
Attribute
  ↓
Aggregate
  ↓
Route
  ↓
Observe
```

Not every event needs every stage synchronously.

A fast acknowledgement path should be separated from slower derived work wherever requirements permit.

## 10. Asynchronous boundaries

Queue/event-bus semantics should be used when a downstream operation is:

- retryable;
- burst-sensitive;
- computationally expensive;
- externally dependent;
- independently scalable;
- not required for synchronous ingestion acknowledgement.

Examples:

- enrichment;
- sessionization;
- attribution recalculation;
- analytics aggregation;
- Meta/Google/TikTok delivery;
- webhook delivery;
- anomaly detection.

The specific broker is deferred until load characteristics and operational requirements are measured.

## 11. Transactional persistence

A transactional relational store should hold Control Plane state and authoritative low-volume operational entities.

Conceptual responsibilities:

```text
Users
Organizations
Workspaces
Members
Sites
Configurations
Integrations
API keys / credential metadata
Orders / lifecycle summaries
Rules / alerts
Dashboard configuration
```

PostgreSQL-compatible storage is a strong candidate because the domain contains relationships, authorization scope, transactional updates and configuration state.

The exact database product is intentionally not frozen by this document.

## 12. Event and analytical persistence

High-volume event facts and analytical workloads may require storage optimized for append-heavy ingestion and columnar/analytical querying.

A separation is permitted:

```text
Transactional / control store
          │
          └── authoritative configuration + operational state

Event / analytical store
          │
          └── canonical event history + analytical projections
```

This does not create two truths. The canonical event fact remains the source; analytical storage is a physical projection optimized for querying.

A ClickHouse-class analytical store is a candidate for this workload, but the choice remains an architecture decision to validate against expected volume, cost and operational complexity.

## 13. Analytics layer

Analytics consumes canonical facts and derived projections to produce:

- funnels;
- campaign/source performance;
- conversion rates;
- revenue;
- refunds;
- costs;
- fees;
- commissions;
- ad spend;
- ROAS;
- profit;
- cohorts;
- data-quality metrics;
- tracking health.

Analytics queries must not scan operational Control Plane tables indiscriminately on every dashboard request.

## 14. Attribution service

Attribution is isolated as a deterministic processing capability.

Inputs:

- canonical touchpoints;
- canonical conversion/Order facts;
- correlation graph;
- attribution configuration;
- attribution window;
- financial eligibility state.

Outputs:

- attribution result;
- model/version;
- eligible touchpoints;
- credit allocation;
- evidence references;
- explanation metadata.

A recalculation produces a new derived result or projection; it does not rewrite source events.

## 15. Commerce lifecycle projection

The system should maintain a derived operational view of Order lifecycle for responsive UI and integrations.

Example:

```text
Canonical facts
      ↓
Lifecycle projector
      ↓
Order current state
      ↓
Dashboard / routing / alerts
```

The current state is a projection. The lifecycle event history remains available as evidence.

## 16. Integration layer

Integrations are isolated behind adapters.

```text
Canonical event
      ↓
Routing policy
      ↓
Provider adapter
      ↓
Provider request
      ↓
Delivery result
```

Each adapter owns:

- provider authentication;
- provider schema mapping;
- provider-specific identifiers;
- retry semantics;
- response handling;
- rate limits;
- provider error normalization.

No provider schema should become the Trackfy universal event schema.

## 17. Event routing

A routing engine may select destinations based on deterministic policy.

Example:

```text
IF event = purchase
AND order.status = approved
AND integration Meta enabled
THEN route
```

Routing must be auditable. Each delivery should reference the canonical source event and its own delivery state.

## 18. Dashboard and application API

The Dashboard should consume an application API rather than reaching directly into event storage for arbitrary queries.

Conceptual flow:

```text
Browser
  ↓
Next.js/application API
  ↓
Control Plane services
  ↓
Analytics query layer
  ↓
Read-optimized data
```

The dashboard must support:

- executive overview;
- campaign/source analysis;
- sales lifecycle;
- revenue/cost/profit;
- funnels;
- event debugger;
- integration status;
- tracking health;
- organization/workspace administration.

The exact frontend framework is an implementation decision; Next.js is a strong candidate for the control/dashboard surface.

## 19. Event Debugger architecture

The debugger is a read model over event traces and processing metadata.

```text
Canonical events
      +
Processing results
      +
Correlation edges
      +
Delivery results
      ↓
Debugger read model
      ↓
Timeline UI
```

It must never modify canonical events merely because an operator opens or inspects a trace.

Debugger queries should favor indexed/read-optimized data rather than reconstructing a complete trace through expensive ad hoc joins on every page load.

## 20. Tracking health architecture

Tracking health is computed from operational signals.

Inputs may include:

- event volume;
- latency;
- duplicate rate;
- unmatched rate;
- browser/server divergence;
- destination delivery failures;
- missing identifiers;
- broken lifecycle relationships.

The resulting health indicators are derived metrics and must link back to evidence.

## 21. Multi-tenancy boundary

Every tenant-owned resource must resolve to an authorization boundary.

Conceptually:

```text
User
 ↓
Organization Membership
 ↓
Organization
 ↓
Workspace
 ↓
Site / Integration / Data
```

The architecture must prevent cross-tenant access in:

- API requests;
- event ingestion;
- background jobs;
- analytics queries;
- debugger queries;
- integration delivery;
- storage access;
- logs and observability where customer data appears.

Authorization belongs to the service responsible for the protected resource; UI filtering is never sufficient.

## 22. Credential boundary

Secrets and credentials follow a strict separation:

```text
Browser
  │
  └── publishable/site-scoped identifier only

Server / Integration runtime
  │
  └── secret credentials

Control Plane
  │
  └── metadata + encrypted/managed secret references
```

Provider secrets must never be returned to browser clients or embedded into public tracking bundles.

## 23. Reliability model

The system should favor at-least-once delivery with explicit deduplication/idempotency rather than assuming exactly-once behavior across distributed boundaries.

Required properties:

- safe retry;
- durable acknowledgement boundaries;
- replayable processing;
- dead-letter/error handling for non-processable events;
- observable delivery state;
- deterministic derived recomputation.

Exactly-once business effects may be achieved at specific boundaries using idempotency, but the overall distributed system should not depend on an impossible global exactly-once assumption.

## 24. Failure isolation

Failures must be compartmentalized.

```text
Meta unavailable
   ≠
Canonical tracking unavailable

Analytics query slow
   ≠
Ingestion unavailable

Dashboard unavailable
   ≠
Event collection unavailable
```

The ingestion path should remain operational when non-critical downstream consumers fail.

## 25. Observability

Every major boundary should expose:

- structured logs;
- metrics;
- traces/correlation IDs;
- processing counters;
- latency;
- error categories;
- retry counts;
- queue/backlog depth where applicable.

The canonical `event_id` should be usable as a cross-component correlation identifier without exposing sensitive payloads unnecessarily.

## 26. Security model

Security controls must include:

- tenant-scoped authorization;
- short-lived or scoped credentials where appropriate;
- secret isolation;
- request validation;
- rate limiting;
- abuse protection;
- auditability for administrative changes;
- restricted raw-event access;
- encrypted transport;
- encryption at rest where provided by the underlying platform;
- privacy-aware logging.

Detailed threat modeling and compliance requirements are separate engineering/security specifications.

## 27. Technology direction

Technology choices are constrained by workload shape rather than preference.

### Candidate application/data services

- **Go:** strong candidate for the high-throughput tracking/data plane, ingestion APIs and workers.
- **Next.js:** strong candidate for dashboard/control-plane application UI and application APIs where appropriate.
- **PostgreSQL/Supabase:** strong candidate for transactional/control-plane state and tenant/RBAC configuration.
- **ClickHouse-class analytics storage:** candidate for large-scale analytical/event querying.
- **Object storage:** candidate for archival/raw evidence or exports where required.
- **Queue/event bus:** required conceptually; concrete technology deferred.

These are candidates, not irreversible architecture commitments.

## 28. Deployment evolution

The initial deployment should not create unnecessary distributed complexity.

A reasonable evolution path is:

```text
Phase A
Control/API + ingestion + relational store

        ↓ measured scale

Phase B
Dedicated async processing + analytical store

        ↓ measured scale

Phase C
Independent tracking edge + workers + specialized consumers
```

Premature microservice decomposition is explicitly avoided. Logical boundaries must exist before physical service boundaries.

## 29. Architectural invariants

1. Canonical events are immutable evidence.
2. Derived projections can be rebuilt.
3. Control Plane is not the event hot path.
4. Dashboard traffic cannot starve ingestion.
5. External providers remain behind adapters.
6. Tenant scope is part of every protected data path.
7. Integration failure cannot delete or invalidate canonical facts.
8. Attribution is reproducible from facts + configuration.
9. Analytics is derived, not authoritative source data.
10. Operational simplicity is preferred until measurements justify additional infrastructure.

## 30. Deferred architecture decisions

The following remain intentionally unresolved:

- exact deployment topology;
- exact cloud/platform provider;
- whether/when Supabase is used directly versus PostgreSQL independently;
- exact event broker/queue;
- exact analytical store;
- raw evidence retention implementation;
- partitioning and sharding strategy;
- event archival strategy;
- exact service boundaries;
- caching strategy;
- CDN/edge provider;
- data residency requirements;
- multi-region strategy;
- SLOs and capacity targets.

These decisions require workload estimates, threat modeling and operational constraints before implementation.

## 31. Architecture acceptance criteria

### AC-01 — Plane isolation
A dashboard workload cannot require a synchronous scan of the high-volume ingestion path to render ordinary analytics.

### AC-02 — Canonical evidence
A valid accepted event remains retrievable and traceable independently of downstream processing success.

### AC-03 — Replay
A derived projection can be rebuilt from retained canonical evidence plus versioned processing configuration.

### AC-04 — Provider isolation
Adding or changing a provider adapter does not require changing the universal canonical event contract unless a genuine domain capability is introduced.

### AC-05 — Tenant isolation
An authorized request for one Organization/Workspace cannot retrieve another tenant's event, analytics or debugger data.

### AC-06 — Failure isolation
A downstream integration outage does not block canonical ingestion beyond documented backpressure limits.

### AC-07 — Attribution determinism
The same canonical facts and attribution configuration produce the same attribution result.

### AC-08 — Observable delivery
Every downstream delivery attempt can be correlated to its canonical source event and inspected for state/outcome.

## 32. Next step

The next specification should define **Multi-tenancy + RBAC v0.1**, including Organization/Workspace boundaries, membership, roles, permission matrix, API-key scopes, isolation rules and authorization invariants.