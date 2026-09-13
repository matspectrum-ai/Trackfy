# Trackfy — Architecture v0.2

**Status:** Draft / Architecture Direction
**Version:** 0.2
**Supersedes:** `docs/architecture/architecture-v0.1.md`

This document consolidates the logical architecture and the current technology direction for Trackfy. It preserves the domain and contract boundaries established by the Product Spec, Domain Model, Event/Tracking Contracts, Attribution Model, Multi-tenancy/RBAC and MVP Spec.

It is an architecture direction, not a deployment manifest or implementation tutorial. Concrete infrastructure sizing, schema details, OpenAPI definitions and production topology are later artifacts.

## 1. Architectural objective

Trackfy is a multi-tenant tracking, attribution and revenue-intelligence platform. The system must accept high-volume tracking traffic without coupling the ingestion hot path to dashboard workloads, tenant administration or third-party delivery.

The architecture is therefore organized around five logical capabilities:

```text
                         Trackfy
                            │
          ┌─────────────────┼─────────────────┐
          │                 │                 │
    Control Plane        Data Plane        Analytics
          │                 │                 │
          └─────────────────┼─────────────────┘
                            │
                      Integrations
                            │
                        Dashboard
```

Logical boundaries come first. Physical services are introduced only when scale, reliability or ownership requires them.

## 2. Technology decisions

The following technologies are the current project direction.

| Area | Decision | Rationale |
|---|---|---|
| Web application | Next.js 16.x + React + TypeScript | Product UI, dashboard and control-plane surface |
| Backend/data plane | Go 1.27.x | High-concurrency ingestion, APIs and workers |
| Transactional database | PostgreSQL 18.x | Relational domain, tenant state, RBAC and transactional consistency |
| Analytics database | ClickHouse | High-volume analytical/event workloads |
| Cache/coordination | Redis | Cache, rate limiting, transient state and initial asynchronous processing |
| Authentication | Better Auth | Application-owned authentication and organization-aware access model |
| Browser tracking | TypeScript Trackfy SDK | Native first-party collection and canonical event production |
| Tag management | Google Tag Manager Web + Server | Compatibility and operational flexibility |
| Object storage | S3-compatible | Raw archives, exports and large objects when required |
| Observability | OpenTelemetry + Prometheus + Grafana | Metrics, traces and operational visibility |
| CI | GitHub Actions | Automated verification and delivery |
| Containers | OCI-compatible containers | Reproducible local and hosted environments |

Current-version references are intentionally kept at the major/minor level in repository documentation so routine patch updates do not create unnecessary architecture churn. As of this architecture decision, Go 1.27.1 is the current 1.27 patch release, and PostgreSQL 18.6 is the current supported 18.x minor release. Next.js 16.3.3 is an active-LTS release as of the current architecture review. These versions must be revalidated before implementation pinning.

## 3. Explicitly excluded platform dependency

Trackfy does not use Supabase as a required platform dependency.

PostgreSQL is treated as a first-class database owned and operated by the project deployment. Authentication, application services and tenant authorization remain under Trackfy's architectural control.

This does not prohibit using managed PostgreSQL infrastructure; it prohibits coupling the domain to a Supabase-specific platform model.

## 4. Logical component map

```text
Customers / Websites / Apps
          │
     SDK / GTM / S2S
          │
          ▼
┌─────────────────────────────┐
│ Tracking Edge / Ingestion   │
│ Go                          │
└──────────────┬──────────────┘
               │
        auth / limits / parse
               │
               ▼
┌─────────────────────────────┐
│ Raw Evidence Boundary       │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│ Canonical Event Pipeline    │
│ Go                          │
└──────────────┬──────────────┘
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
 Identity   Enrichment  Sessionization
 Correlation Normalize
     │         │         │
     └─────────┼─────────┘
               ▼
┌─────────────────────────────┐
│ Derived Processing          │
│ lifecycle / attribution     │
└──────────────┬──────────────┘
               │
      ┌────────┼────────┐
      ▼        ▼        ▼
 Attribution Analytics Lifecycle
      │        │        │
      └────────┼────────┘
               ▼
      Integration Router
               │
     ┌─────────┼─────────┐
     ▼         ▼         ▼
   Meta     Google     TikTok
   /CAPI     /Ads      /Events
               │
               ▼
       CRM / Webhooks

       CONTROL PLANE
┌─────────────────────────────┐
│ Next.js + Go + PostgreSQL   │
│ users / orgs / workspaces   │
│ RBAC / sites / integrations │
│ configs / dashboards        │
└──────────────┬──────────────┘
               ▼
           Dashboard
```

## 5. Control Plane

The Control Plane manages tenant-owned configuration and operational state.

Responsibilities:

- users and sessions;
- organizations and memberships;
- workspaces;
- roles and permissions;
- sites and tracking configuration;
- integrations and destination configuration;
- API keys and credential metadata;
- attribution configuration;
- campaign/ad-account metadata;
- dashboards, reports, rules and alerts;
- billing and entitlement state when introduced.

The Control Plane is not the ingestion hot path.

## 6. Data Plane

The Data Plane handles event collection and processing.

Responsibilities:

- Trackfy Browser SDK ingestion;
- GTM Web ingestion;
- GTM Server ingestion;
- authenticated S2S APIs;
- provider webhooks;
- offline/backfill ingestion;
- transport and contract validation;
- tenant/site resolution;
- deduplication and idempotency;
- raw evidence preservation;
- canonical event creation;
- correlation and enrichment;
- asynchronous processing;
- downstream routing and delivery.

The Data Plane must scale independently of dashboard traffic.

## 7. Application boundaries

The system should initially remain a small number of deployable applications with strong internal modules rather than immediately becoming a microservice fleet.

Recommended initial logical units:

```text
trackfy-web
  Next.js dashboard / application UI

trackfy-api
  Go application API / domain services

trackfy-ingest
  Go ingestion path

trackfy-worker
  Go asynchronous processors
```

These may initially share a repository and deployment boundaries. They should communicate through explicit contracts even when deployed together.

A separate deployable is justified only when an operational requirement exists.

## 8. PostgreSQL

PostgreSQL is the transactional system for low-to-moderate volume relational state.

Primary responsibilities:

```text
users
organizations
memberships
roles / permissions
workspaces
sites
tracking configuration
integrations
credential metadata
API keys
orders / lifecycle projections
rules / alerts
dashboards / reports
billing / entitlements
```

PostgreSQL is also appropriate for transactional idempotency records and operational job state when bounded by documented retention and scale assumptions.

It is not the default analytical event warehouse.

## 9. ClickHouse

ClickHouse is the analytical/event-storage direction for high-volume append-heavy data and analytical queries.

Responsibilities may include:

- canonical event history optimized for analytics;
- event timelines;
- funnel analysis;
- campaign/source performance;
- attribution analysis;
- revenue/cost/profit analytics;
- tracking-health calculations;
- large time-window queries.

ClickHouse remains a projection of canonical facts, not a competing source of truth.

The introduction threshold is operational rather than ideological. MVP may begin with a simpler analytical path if measured volume does not justify immediate ClickHouse deployment.

## 10. Redis

Redis has a deliberately constrained role:

- cache;
- rate limiting;
- short-lived coordination;
- transient processing state;
- idempotency acceleration where safe;
- initial streams/queues for asynchronous work.

Redis must not become the authoritative store for events, Orders or tenant configuration.

A dedicated broker may replace or complement Redis Streams when throughput, delivery guarantees or operational requirements justify it.

## 11. Authentication and authorization

Authentication is application-owned.

```text
Next.js
   ↓
Better Auth
   ↓
PostgreSQL
```

Authorization follows the Multi-tenancy/RBAC specification:

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

Tenant isolation is enforced server-side in every protected path.

UI filtering is not considered authorization.

## 12. Tracking collection

Trackfy supports four principal collection paths:

```text
Browser SDK ─────┐
GTM Web ─────────┤
GTM Server ──────┤
S2S / Webhook ───┤
Offline ──────────┘
        ↓
   Go ingestion
        ↓
 canonical events
```

### Browser SDK

Creates or reuses permitted browser identity, captures acquisition metadata and emits canonical events to a first-party/Trackfy endpoint.

### GTM Web

Acts as a compatible orchestration layer mapping Data Layer state into the Trackfy event contract.

### GTM Server

Provides a server-side collection path and provider adapter boundary.

### S2S/Webhook

Provides authoritative backend/provider facts such as Order and payment lifecycle transitions.

## 13. First-party tracking

Trackfy should support customer-controlled tracking origins where technically and legally appropriate.

```text
customer domain
      ↓
tracking.customer-domain.example
      ↓
Trackfy ingestion
```

First-party collection is a transport architecture, not a guarantee against browser restrictions or consent requirements.

## 14. Ingestion hot path

The ingestion path must be optimized for a fast, durable acknowledgement.

```text
receive
  ↓
authenticate / resolve scope
  ↓
validate transport
  ↓
assign received_at
  ↓
compute dedupe/idempotency decision
  ↓
preserve evidence
  ↓
accept canonical event
  ↓
acknowledge
```

Expensive analytics, enrichment, attribution and third-party delivery should be asynchronous unless an explicit synchronous requirement exists.

## 15. Canonical event storage strategy

The source-of-truth concept is logical, not tied to one physical database.

```text
Raw evidence
    ↓
Canonical event
    ↓
Derived projections
```

The canonical event history must remain replayable. A physical storage layout may use PostgreSQL, ClickHouse and/or object storage at different scales, provided the original fact remains traceable and versioned.

## 16. Asynchronous processing

An asynchronous boundary is used where work is retryable, burst-sensitive, expensive or externally dependent.

Initial worker categories:

```text
enrichment
correlation
sessionization
attribution
analytics projection
integration delivery
webhook delivery
health/anomaly processing
```

Workers must carry tenant context and source event IDs through processing.

## 17. Attribution

Attribution is a deterministic derived capability.

```text
Canonical facts
      ↓
eligible touchpoints
      ↓
attribution model
      ↓
credit allocation
      ↓
attribution result
```

Recalculation creates or replaces a derived projection while preserving the underlying evidence.

## 18. Lifecycle and financial projections

Orders and payment lifecycle facts remain evidence-driven.

```text
canonical lifecycle events
          ↓
     lifecycle projector
          ↓
   current Order state
          ↓
 analytics / routing / UI
```

Revenue calculations distinguish gross, pending, approved, refunds, fees, commissions, ad spend and profit according to the financial contract.

## 19. Integration architecture

External platforms are adapters behind an internal routing contract.

```text
canonical event
      ↓
routing policy
      ↓
provider adapter
      ↓
provider request
      ↓
delivery state
```

No provider schema becomes Trackfy's canonical schema.

Adapters must retain enough outbound metadata to debug delivery without exposing secrets.

## 20. Analytics architecture

Dashboard analytics should query read-optimized representations rather than repeatedly scanning operational tables.

Conceptual model:

```text
Canonical events
      ↓
analytics projections
      ↓
metrics / dimensions
      ↓
application API
      ↓
Next.js dashboard
```

The dashboard must support the MVP's required views without coupling UI components directly to storage schemas.

## 21. Event Debugger

The debugger is a read model over:

- canonical events;
- correlation edges;
- processing outcomes;
- lifecycle projections;
- downstream delivery results;
- quality signals.

It should expose a coherent journey:

```text
Click → Session → Page View → Lead → Checkout → Order → Status → Refund
```

Opening a trace must never mutate the underlying facts.

## 22. Tracking Health

Tracking Health is derived from operational evidence:

- event volume anomalies;
- latency;
- duplicate rate;
- unmatched rate;
- missing identifiers;
- browser/server divergence;
- lifecycle inconsistencies;
- destination failures.

Health indicators must be explainable back to evidence.

## 23. Reliability model

The distributed system assumes at-least-once delivery with explicit idempotency and deduplication.

Required properties:

- durable acknowledgement boundaries;
- safe retries;
- replayable processing;
- bounded dead-letter/error handling;
- observable processing state;
- deterministic recomputation of derived state.

The architecture does not assume global exactly-once delivery.

## 24. Failure isolation

Subsystem failures must not cascade unnecessarily.

```text
Meta outage
  ≠ canonical event loss

ClickHouse degradation
  ≠ ingestion failure

Dashboard outage
  ≠ tracking outage

One adapter failure
  ≠ all destination delivery failure
```

The ingestion path is the highest-priority availability path.

## 25. Security boundaries

Secrets are separated by trust level.

```text
Browser
  → publishable/site-scoped identifier only

Server runtime
  → secret credentials

Control Plane
  → configuration + managed secret references
```

Required controls include tenant authorization, rate limiting, payload validation, secret isolation, secure transport, restricted raw-event access and privacy-aware logs.

## 26. Observability

OpenTelemetry provides common tracing and metric instrumentation.

Every important request/job should carry:

- correlation ID;
- tenant context where safe;
- canonical event ID when applicable;
- latency;
- outcome;
- retry count;
- error class.

Prometheus/Grafana are the initial operational analysis surface.

Customer-visible Event Debugger and internal observability are related but distinct concerns.

## 27. Deployment direction

Initial production deployment should favor operational simplicity:

```text
                 Internet
                    │
                edge/proxy
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
     Next.js web          Go ingestion/API
                              │
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                PostgreSQL  Redis    ClickHouse*
                    │         │         │
                    └─────────┼─────────┘
                              ▼
                           Workers
```

`*` ClickHouse may be introduced from the beginning or at the measured scale threshold defined by the MVP validation plan.

Next.js may be deployed on Vercel or another compatible platform. Go services must run on infrastructure appropriate for persistent, high-throughput server workloads.

## 28. Local development

The repository should support a reproducible containerized development environment for:

```text
Next.js
Go
PostgreSQL
Redis
ClickHouse (when enabled locally)
observability tooling where needed
```

The developer workflow should remain usable on modest hardware by allowing individual dependencies to be started selectively.

## 29. Architectural invariants

1. Canonical events are immutable evidence.
2. Derived state is rebuildable.
3. PostgreSQL is the transactional control-plane store, not the universal analytics engine.
4. Redis is never the source of truth.
5. ClickHouse is a read/analytics projection, not a second domain truth.
6. Control Plane does not become the tracking hot path.
7. Tenant context is enforced beyond the UI.
8. Provider schemas remain behind adapters.
9. Integration failure cannot erase canonical facts.
10. Authentication and authorization remain distinct concerns.
11. Contracts remain technology-independent.
12. Physical service decomposition follows measured need.
13. Browser clients never receive provider secrets.
14. Every asynchronous boundary remains observable.
15. Derived financial and attribution values remain reproducible from source evidence.

## 30. Decisions intentionally deferred

- final hosting providers for Go/PostgreSQL/ClickHouse;
- exact queue technology beyond initial Redis capability;
- exact ClickHouse introduction threshold;
- database partitioning and retention strategy;
- exact API gateway/edge vendor;
- secret-management provider;
- production object-storage provider;
- infrastructure-as-code tool;
- Kubernetes adoption, if ever required;
- exact autoscaling policy;
- final schema/index design.

These are implementation and operations decisions, not reasons to reopen the domain architecture.

## 31. Architecture acceptance criteria

### AC-01 — Tenant isolation
No protected request, job or query may access data outside its authorized Organization/Workspace scope.

### AC-02 — Hot-path isolation
Dashboard or analytics load must not be required for successful event acknowledgement.

### AC-03 — Replayability
Accepted canonical events must remain sufficient to reconstruct supported derived state within the documented retention window.

### AC-04 — Provider isolation
Adding or changing a provider adapter must not require changing the canonical event semantics unless the domain itself changes.

### AC-05 — Failure isolation
A destination outage must not invalidate or duplicate canonical events.

### AC-06 — Stack independence
No product-critical domain behavior may depend on a Supabase-specific API or platform abstraction.

### AC-07 — Reproducibility
Attribution and analytical projections must be reproducible from canonical facts plus versioned configuration.

### AC-08 — Observability
Every accepted event must be traceable through ingestion and downstream processing using canonical identifiers.

## 32. Next step

With the architecture direction fixed, the next artifact is the **First Implementation Slice Spec v0.1**.

That slice should prove the architecture end-to-end with the smallest useful vertical path:

```text
tenant
  ↓
site
  ↓
first-party ingestion
  ↓
canonical event
  ↓
PostgreSQL persistence
  ↓
processing
  ↓
query/read model
  ↓
Event Debugger
```

No broad feature expansion should begin until this slice is verified.