# Trackfy — Event & Tracking Contracts v0.2

**Status:** Draft
**Version:** 0.2
**Scope:** Browser-side tracking, Trackfy Browser SDK, Google Tag Manager Web, server-side GTM compatibility, first-party collection, server-to-server ingestion, webhooks, offline/backfill ingestion, identity continuity, acquisition metadata, event normalization, enrichment, deduplication, idempotency, correlation, routing, data quality and observability.

This document extends v0.1. It defines the semantic boundary between Trackfy producers/integrations and the canonical tracking pipeline. It is implementation-neutral: OpenAPI, JSON Schema, storage schemas and infrastructure are separate engineering artifacts.

## 1. Contract goals

Trackfy must be able to determine:

1. What happened.
2. When it happened.
3. Where it came from.
4. Which Organization, Workspace and Site own it.
5. Which Visitor, Session and acquisition path it can be correlated with.
6. Whether it was delivered more than once.
7. Whether it was accepted, rejected, duplicated, unmatched, processed or failed.
8. Which external provider or system asserted the fact.
9. Whether raw evidence can be replayed without rewriting history.
10. Whether a conversion or Order can be connected to acquisition evidence.
11. Whether a canonical event can be safely distributed downstream.
12. Whether tracking quality or conversion-path integrity degraded.

The contract prioritizes immutable evidence, deterministic correlation, explicit provenance and replayability.

## 2. Collection architecture

Trackfy supports multiple producers feeding one canonical pipeline:

```text
Browser SDK ──────────────┐
GTM Web ──────────────────┤
GTM Server ───────────────┤
Server / S2S API ─────────┤
Provider Webhook ─────────┤
Offline / Backfill ───────┤
                         ▼
                  Ingestion Gateway
                         │
              Authentication / Limits
                         │
                  Raw Evidence Layer
                         │
                         ▼
                 Canonical Event Layer
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
    Validation       Identity          Enrichment
                     Correlation       / Normalize
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                  Derived Processing
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   Sessionization    Attribution      Lifecycle
        │                │                │
        └────────────────┼────────────────┘
                         ▼
                  Routing / Fan-out
                         │
   Meta / Google / TikTok / GA4 / CRM / Webhooks
                         │
                         ▼
                 Quality / Debugger
```

An ingestion request is not itself the canonical business fact. An accepted canonical event is durable evidence; processing, attribution, analytics and delivery states are derived.

## 3. Collection modes

### 3.1 Trackfy Browser SDK

The native SDK is the preferred low-friction browser producer.

It may:

- establish or reuse permitted first-party Visitor identity;
- establish or reuse Session context;
- capture acquisition metadata;
- generate `event_id` before transmission;
- emit standard and custom events;
- queue/retry transient failures within documented limits;
- expose client diagnostics without exposing server secrets.

The SDK is a producer, not an authorization source.

### 3.2 Google Tag Manager Web

Trackfy must support GTM Web as a collection and tag-orchestration source.

```text
Website → GTM Web → Trackfy ingestion
```

The integration must define a stable mapping from GTM Data Layer events and variables to the Trackfy canonical envelope.

Browser-visible GTM configuration must never contain Trackfy server credentials.

### 3.3 Server-side GTM

Trackfy must accept events produced by a server-side GTM container.

```text
Browser / GTM Web → GTM Server Container → Trackfy
```

The GTM server boundary is an authenticated producer/integration, not a separate Trackfy source-of-truth model.

### 3.4 Server-to-server API

Trusted backends may emit events directly for authoritative or browser-independent facts, including Orders, lifecycle transitions, CRM conversions, refunds, postbacks and offline conversions.

### 3.5 Webhooks

Provider callbacks are ingested through authenticated webhook endpoints. Provider event IDs, types, source metadata and receipt metadata must be preserved before normalization.

### 3.6 Offline and backfill

Trackfy must support controlled delayed ingestion of historical or offline facts. Delayed arrival does not convert `occurred_at` into ingestion time.

## 4. Canonical event envelope

Conceptual envelope:

```json
{
  "event_id": "evt_01J...",
  "event_name": "purchase",
  "event_version": 1,
  "occurred_at": "2026-09-13T06:30:00.123Z",
  "received_at": "2026-09-13T06:30:00.456Z",
  "source": {
    "type": "browser|server|gtm_web|gtm_server|webhook|offline|api",
    "provider": "trackfy_js",
    "integration_id": null
  },
  "tenant": {
    "organization_id": "org_...",
    "workspace_id": "ws_...",
    "site_id": "site_..."
  },
  "identity": {
    "visitor_id": "vis_...",
    "session_id": "ses_...",
    "click_id": "clk_...",
    "customer_id": null,
    "external_customer_id": null,
    "external_order_id": null
  },
  "acquisition": {
    "utm_source": "meta",
    "utm_medium": "paid_social",
    "utm_campaign": "campaign-a",
    "utm_content": "creative-1",
    "utm_term": null,
    "fbclid": "...",
    "gclid": null,
    "ttclid": null,
    "msclkid": null,
    "twclid": null,
    "li_fat_id": null,
    "referrer": "https://..."
  },
  "context": {
    "url": "https://example.com/landing",
    "path": "/landing",
    "title": "Landing",
    "device_type": "desktop",
    "locale": "pt-BR",
    "timezone": "America/Santarem"
  },
  "consent": {},
  "payload": {},
  "provenance": {
    "external_event_id": null,
    "source_record_id": null,
    "raw_event_reference": null
  }
}
```

Field names remain provisional until machine-readable schemas are introduced. The semantic rules here govern v0.2.

## 5. Identity contract

### `event_id`

Uniquely identifies one canonical event assertion. It is opaque, immutable, collision-resistant and never reused for another material event.

### `visitor_id`

Represents a measurement identity within the permitted tracking model. It does not prove human identity.

### `session_id`

Groups interactions according to the canonical sessionization policy. A client may propose the identifier; the server validates scope and relationships.

### `click_id`

Trackfy's internal acquisition-click identity. Provider IDs such as `fbclid`, `gclid` and `ttclid` remain separate signals.

### Customer and Order identity

Customer and Order identifiers are distinct from Visitor/Session identity. External identifiers always carry provider/integration scope.

## 6. Acquisition contract

Raw acquisition evidence must be preserved before reporting normalization.

Canonical dimensions include:

```text
utm_source
utm_medium
utm_campaign
utm_content
utm_term
fbclid
gclid
ttclid
msclkid
twclid
li_fat_id
referrer
click_id
```

UTM values are caller-provided evidence and must not be silently rewritten. Normalized reporting dimensions are derived values.

Missing referrer or click identifiers are valid states and must not be interpreted as proof of direct traffic.

Provider click identifiers are correlation signals, not replacements for UTM dimensions.

## 7. First-party collection

Trackfy should support a customer-controlled tracking origin where technically and legally appropriate.

```text
customer-domain.example
        ↓
tracking.customer-domain.example
        ↓
Trackfy ingestion
```

The contract distinguishes:

- browser-observed values;
- Trackfy transport metadata;
- provider-reported assertions.

First-party collection does not remove browser restrictions, consent requirements or uncertainty.

## 8. Cross-domain continuity

Trackfy should support explicit continuity across related domains/subdomains.

```text
landing.example.com
        ↓
checkout.example.com
        ↓
pay.example.com
```

Continuity requires an intentional identity-transfer mechanism. Trackfy must not silently merge identities based only on similarity.

## 9. Event taxonomy

The standard taxonomy remains intentionally compact.

### Acquisition / navigation

- `page_view`
- `landing_view`
- `click`
- `engagement`

### Intent / conversion

- `form_start`
- `form_submit`
- `lead`
- `checkout_started`
- `checkout_completed`

### Commerce

- `order_created`
- `order_status_changed`
- `purchase`
- `refund`
- `chargeback`

### Transport / operations

- `postback_received`
- `integration_event_received`
- `tracking_error`

Custom events are allowed but must use the canonical envelope and must not replace a required business lifecycle event.

## 10. Browser tracking behavior

```text
Page / interaction
      ↓
SDK or GTM Web
      ↓
Identity + acquisition context
      ↓
Generate event_id
      ↓
First-party / Trackfy endpoint
      ↓
Server validation
      ↓
Canonical acceptance
```

Browser payloads are untrusted. Trackfy assigns `received_at`, enforces authorization and tenant scope, validates size/shape and applies abuse controls.

The system must tolerate duplicate, delayed, abandoned and out-of-order browser delivery.

## 11. GTM contracts

### GTM Web

Trackfy must define:

- accepted Data Layer event names;
- event-to-envelope mappings;
- variable mappings;
- `event_id` generation/preservation;
- consent gating behavior;
- failure/retry behavior;
- debugger visibility.

### GTM Server

Trackfy must define:

- server authentication;
- GTM request-to-event mapping;
- preservation of client/provider identifiers;
- idempotency behavior;
- response semantics;
- downstream routing semantics.

GTM remains an integration mechanism; Trackfy remains the canonical tracking system.

## 12. Server-side and S2S contract

Server-originated events are preferred for authoritative business outcomes and browser-independent lifecycle transitions.

Every trusted producer must be scoped to an Organization/Workspace/Integration using tenant-safe credentials.

No server credential may be embedded in a browser-facing SDK or GTM Web tag.

## 13. Raw evidence and normalization

```text
Raw external assertion
        ↓
Verified / accepted transport
        ↓
Normalization
        ↓
Canonical event
        ↓
Derived state
```

The raw/provider assertion must remain traceable after normalization. Provider-specific fields belong primarily to adapter/provenance layers, not to the universal canonical contract.

## 14. Enrichment

Deterministic post-ingestion enrichment may add:

- Visitor/Session/Click references;
- Campaign/source dimensions;
- request/device context;
- provider metadata;
- Customer/Order relationships;
- integration metadata;
- permitted geography/device dimensions;
- correlation quality.

Enrichment is derived metadata. It must never overwrite source evidence.

## 15. Deduplication

### Canonical event deduplication

The same `event_id` must result in one canonical event regardless of delivery count.

```text
first delivery → accepted
repeat delivery → duplicate / replay
```

Duplicate deliveries remain observable.

### Provider deduplication

When a provider offers a stable event ID, use that ID within provider/integration scope. Unsafe timestamp/amount heuristics must not be the primary key when stable provider identity exists.

Browser/server copies of the same conversion must have an explicit shared logical identity or provider-supported deduplication key.

## 16. Idempotency

Deduplication protects event identity; idempotency protects retry effects.

For mutating APIs:

- identical retries with the same idempotency key return the same logical outcome;
- materially different requests using an existing idempotency key are rejected;
- scope is tenant/integration aware;
- retention covers the documented retry window;
- retries cannot create duplicate Orders, refunds or lifecycle transitions.

## 17. Correlation and identity graph

Default deterministic preference:

```text
Explicit Trackfy identity
        ↓
Provider/external ID + integration scope
        ↓
click_id / provider click ID
        ↓
session_id
        ↓
visitor_id
        ↓
other configured deterministic signal
        ↓
unmatched
```

Potential graph:

```text
Visitor → Session → Click → Lead → Customer → Order
```

Every relationship should retain an evidence class. Heuristic similarity cannot silently become a canonical identity edge.

## 18. Attribution recovery boundary

The tracking pipeline must permit downstream attribution to recover conversion relationships when direct browser context is missing but deterministic evidence exists.

Example:

```text
Purchase
  ↓
no session_id
  ↓
customer_id + provider click ID
  ↓
deterministic correlation
  ↓
recovered acquisition path
```

Recovered relationships must remain distinguishable from direct Session/Click relationships.

## 19. Commerce lifecycle contract

Tracking milestones and commerce facts remain separate.

```text
click
 ↓
page_view
 ↓
lead
 ↓
checkout_started
 ↓
order_created
 ↓
order_status_changed: pending → approved
 ↓
refund
```

`purchase` does not automatically imply `approved` unless explicitly defined by the source contract.

Provider lifecycle assertions retain source/provider provenance.

## 20. Routing and fan-out

One canonical event may feed multiple destinations:

```text
Canonical purchase
       │
       ├── Meta
       ├── Google Ads
       ├── GA4
       ├── TikTok
       ├── CRM
       └── Custom webhook
```

Routing decisions are rule-driven and independently observable.

Example:

```text
IF event = purchase
AND order.status = approved
THEN route to configured conversion destinations
```

A destination failure must not turn the canonical business fact into a failed business event.

## 21. Downstream adapter contract

Every delivery record should retain:

- destination/provider;
- destination account/property;
- source canonical `event_id`;
- outbound request ID;
- delivery state;
- retry state;
- provider request/response metadata where safe;
- provider-side event ID when available.

Adapters translate Trackfy events into provider-specific contracts. Provider-specific semantics should not contaminate the canonical event model.

## 22. Data quality and tracking health

Trackfy must derive measurable quality signals from canonical facts.

Examples:

```text
browser/server divergence
missing click IDs
missing session IDs
purchase without order
authoritative order without attribution
duplicate rate
unmatched rate
integration failure rate
event-volume anomalies
latency anomalies
```

The product should support threshold-based detection and alerts without treating the health score as raw tracking evidence.

## 23. Processing states

Initial conceptual states:

- `accepted`
- `duplicate`
- `rejected`
- `processed`
- `failed`
- `unmatched`
- `routed`
- `delivery_failed`

Processing states are operational. They are not Commerce lifecycle states.

## 24. Validation layers

Transport validation covers authentication, size, content type, rate limits and protocol validity.

Contract validation covers required fields, types, versions and payload structure.

Tenant validation covers Organization, Workspace, Site and credential scope.

Semantic validation covers event-specific invariants.

Correlation validation covers structural relationship integrity.

Destination validation covers whether a downstream adapter has sufficient data to build a compliant request.

Rejections must expose machine-readable reasons and, where authorized, debugger-visible traces.

## 25. Consent and privacy boundary

The event contract supports explicit consent/context signals but does not claim that technical receipt constitutes legal consent.

The system must support:

- secret separation from browser code;
- tenant-scoped credentials;
- safe logging;
- data minimization where appropriate;
- retention/deletion controls;
- authorization on raw evidence and debugger traces.

The complete privacy/compliance policy is a separate specification.

## 26. Event Debugger contract

For an authorized event trace, the debugger should show:

```text
Event ID
Event type + version
Occurred at
Received at
Source / provider
Organization / Workspace / Site
Visitor / Session / Click IDs
UTMs + provider click identifiers
Payload summary
Raw/canonical provenance reference
Processing state
Deduplication result
Correlation result
Enrichment result
Routing status
Destination delivery state
Errors / rejection reason
```

The primary visualization is a trace/timeline rather than only an event table.

## 27. Replay and recomputation

Canonical evidence must be replayable to regenerate derived state, including sessions, attribution, lifecycle projections, analytics and permitted routing decisions.

Replay must not produce duplicate business outcomes.

## 28. Contract versioning

Every event type carries `event_version`.

Breaking semantic changes require a new version. Compatible additions may remain within the same version according to the future schema compatibility policy.

Replay preserves original event version. Derived processors record the version consumed.

## 29. Acceptance criteria

### AC-01 — Canonical identity
An accepted event has immutable identity and preserved provenance.

### AC-02 — Browser/server deduplication
The same logical conversion arriving from browser and server does not become two canonical events when they share the defined identity key.

### AC-03 — GTM Web
A supported GTM Web mapping produces canonical Trackfy events without exposing server credentials.

### AC-04 — GTM Server
A server-side GTM producer authenticates, preserves provenance and obeys idempotency rules.

### AC-05 — First-party collection
A supported first-party origin can submit browser events while Trackfy remains authoritative for receipt time and authorization.

### AC-06 — Cross-domain continuity
A configured identity-transfer flow can preserve Visitor/Session continuity across related domains without heuristic merging.

### AC-07 — Late arrival
Late events preserve `occurred_at`, record Trackfy `received_at` and remain processable.

### AC-08 — Provider provenance
A provider webhook event can be traced to its normalized canonical event.

### AC-09 — Deterministic correlation
An Order with valid deterministic identity evidence can connect to acquisition evidence without requiring a browser session at purchase time.

### AC-10 — Enrichment safety
Derived enrichment cannot overwrite raw acquisition evidence.

### AC-11 — Routing isolation
Downstream delivery failure does not mutate the canonical event into a failed business fact.

### AC-12 — Debuggability
An authorized operator can inspect the path from ingestion to correlation and downstream routing.

### AC-13 — Replay safety
Replay does not generate duplicate Orders, refunds or lifecycle effects.

### AC-14 — Data quality
Duplicate, unmatched, missing-correlation and destination-failure metrics can be derived from canonical facts.

## 30. External compatibility notes

Trackfy's adapter layer should accommodate current external analytics/advertising patterns without making them part of the canonical contract.

For example, Google Analytics 4's Measurement Protocol accepts server-side HTTPS event delivery and uses web-stream `client_id` / app-stream `app_instance_id`; Google also documents event/session timestamping and transport constraints. citeturn779105search1turn779105search2

TikTok documents event deduplication when Pixel and Events API both report the same conversion and requires an event ID for that deduplication path. citeturn779105search6

These are adapter requirements, not reasons to copy external schemas into Trackfy's canonical event model.

## 31. Explicitly deferred

The following remain separate design decisions:

- Browser SDK public API;
- GTM Web tag/template packaging;
- GTM Server deployment topology;
- first-party DNS/proxy implementation;
- cookie/storage mechanics;
- exact session timeout and re-identification rules;
- complete provider identifier catalog;
- consent-management integration;
- schema registry implementation;
- public OpenAPI definitions;
- provider adapter payload schemas;
- routing rule language;
- quality scoring formulas;
- retention periods;
- privacy/compliance policy;
- attribution windows/models;
- analytical storage technology.

## 32. Next step

The next specification is **Attribution Model v0.1**. It will define conversion credit and revenue attribution across first-touch, last-touch and multi-touch models, attribution windows, lifecycle states, refunds, fees, commissions, ad spend and profit.
