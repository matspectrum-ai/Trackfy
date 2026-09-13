# Trackfy — Event & Tracking Contracts v0.1

**Status:** Draft
**Version:** 0.1
**Scope:** Browser-side tracking, server-side ingestion, event identity, deduplication, idempotency, timestamps, acquisition metadata, correlation and downstream normalization.

This document defines the first machine-oriented contract boundary between Trackfy clients/integrations and the canonical tracking pipeline. It is a logical contract, not yet an implementation-specific OpenAPI, JSON Schema or database schema.

## 1. Contract goals

Trackfy must be able to answer, deterministically:

1. What happened?
2. When did it happen?
3. Where did it come from?
4. Which tenant, workspace and site owns it?
5. Which visitor/session/click journey does it belong to?
6. Was the event received more than once?
7. Has the event already been processed?
8. Which external provider or source asserted the fact?
9. Can the same raw evidence be replayed to rebuild derived state?
10. Can a conversion or order be connected back to acquisition evidence without relying on mutable UI state?

The contract therefore prioritizes immutable evidence, deterministic correlation and safe replay over convenience of individual producers.

## 2. Contract boundaries

There are four distinct boundaries:

```text
Browser / Server / Provider
          │
          ▼
   Ingestion Contract
          │
          ▼
 Canonical Event Envelope
          │
          ├── Identity / correlation
          ├── Acquisition metadata
          ├── Business payload
          └── Provenance / processing metadata
          │
          ▼
   Derived processing
          │
          ├── Sessionization
          ├── Matching
          ├── Attribution
          ├── Lifecycle projection
          └── Analytics
```

An ingestion request is not itself the canonical business fact. The accepted canonical event is the durable evidence. Processing results are derived state.

## 3. Canonical event envelope

Every Trackfy event SHOULD conform conceptually to the following envelope:

```json
{
  "event_id": "evt_01J...",
  "event_name": "page_view",
  "event_version": 1,
  "occurred_at": "2026-09-13T06:30:00.123Z",
  "received_at": "2026-09-13T06:30:00.456Z",
  "source": {
    "type": "browser",
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
    "external_customer_id": null
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
  "payload": {},
  "provenance": {
    "external_event_id": null,
    "source_record_id": null
  }
}
```

Field names are provisional until the implementation contract is versioned. The semantic rules in this document are authoritative for v0.1.

## 4. Required versus optional fields

### 4.1 Required for every canonical event

- `event_id`
- `event_name`
- `event_version`
- `occurred_at`
- `received_at`
- `source.type`
- `tenant.organization_id`
- `tenant.workspace_id`
- `tenant.site_id` when the event is site-scoped
- at least one valid correlation or source context appropriate to the event
- `payload` when the event type requires business data

### 4.2 Optional by event/source

- `visitor_id`
- `session_id`
- `click_id`
- UTM dimensions
- provider click identifiers
- `referrer`
- URL/context fields
- external customer/order/payment identifiers
- integration identifiers
- provider-specific payload fields

An event must not be rejected merely because an identity field is unavailable when that field is legitimately impossible to obtain at the source.

## 5. Event identity

### 5.1 event_id

`event_id` is the globally unique identity of one canonical event assertion.

Rules:

- generated before or at ingestion;
- immutable after acceptance;
- opaque to business semantics;
- safe to expose in debugger responses;
- used as the primary event-level deduplication key;
- must not be reused for a materially different event.

Recommended format: opaque, sortable, collision-resistant identifier with a Trackfy prefix, such as `evt_...`. The exact identifier algorithm is an implementation decision.

### 5.2 visitor_id

`visitor_id` represents a Trackfy measurement identity, not a guaranteed human identity.

It may persist across sessions when the configured measurement model permits it. It must not be interpreted as proof that two interactions were performed by the same person.

### 5.3 session_id

`session_id` groups a bounded sequence of interactions under the sessionization policy.

A client MAY create a session identifier locally. The server remains responsible for validating tenant/site scope and applying the canonical sessionization rules.

### 5.4 click_id

`click_id` identifies an acquisition click recognized by Trackfy.

It must be distinct from provider-specific identifiers such as `fbclid`, `gclid` and `ttclid`. A provider identifier may be an input used to create or correlate a Trackfy `click_id`.

### 5.5 external identifiers

Provider identifiers such as external order IDs, customer IDs and event IDs must always carry source/provider context. An identifier string alone is never assumed globally unique across providers.

## 6. Acquisition metadata contract

Trackfy preserves acquisition metadata as evidence rather than immediately collapsing it into one source label.

Canonical fields currently include:

```text
utm_source
utm_medium
utm_campaign
utm_content
utm_term
fbclid
gclid
ttclid
click_id
referrer
```

The model may later add provider-specific or first-party identifiers, but additions must preserve backwards compatibility through contract versioning.

### 6.1 UTM semantics

UTM values are caller-provided acquisition metadata. Trackfy must preserve the received value and should not silently rewrite it during ingestion.

Normalization may exist as a derived representation for reporting, but the raw received value remains available as evidence.

### 6.2 Click identifiers

`fbclid`, `gclid`, `ttclid` and equivalent provider identifiers are correlation signals. They are not interchangeable with UTM fields and must remain separately addressable.

### 6.3 Referrer

Referrer information is contextual acquisition evidence. Missing, suppressed or unavailable referrer data is valid and must not be treated as proof of direct traffic.

## 7. Event taxonomy v0.1

The initial contract distinguishes lifecycle and instrumentation events.

### Acquisition / navigation

- `page_view`
- `click`
- `landing_view`

### Intent / conversion

- `lead`
- `checkout_started`
- `checkout_completed`

### Commerce

- `order_created`
- `order_status_changed`
- `purchase`
- `refund`
- `chargeback`

### Integration / transport

- `postback_received`
- `integration_event_received`
- `tracking_error`

This taxonomy is intentionally small. New event types require an explicit contract change and should not be created merely because a UI screen needs a new label.

## 8. Payload rules

The envelope owns identity, timestamps, tenant context, provenance and common acquisition metadata. Event-specific `payload` owns event semantics.

For example:

```json
{
  "event_name": "purchase",
  "payload": {
    "order_id": "ord_...",
    "external_order_id": "12345",
    "currency": "BRL",
    "gross_amount": 199.90
  }
}
```

Payloads must be versioned with the event contract. Adding an optional field is generally backward compatible; changing field meaning, units or requiredness is a breaking contract change.

Amounts must carry explicit currency. Monetary precision and rounding rules belong to the financial contract, not to arbitrary producer conventions.

## 9. Timestamp contract

Trackfy distinguishes at least:

- `occurred_at`: when the source says the event happened;
- `received_at`: when Trackfy received the event.

These timestamps must never be conflated.

Rules:

1. `occurred_at` must be represented as an absolute timestamp with timezone/UTC normalization.
2. `received_at` is assigned by the receiving system and must not be trusted from a browser payload.
3. Event ordering must use canonical timestamps plus event identity; arrival order alone is insufficient.
4. Late-arriving events are valid and must remain processable.
5. Clock skew must be tolerated within explicit validation boundaries.
6. Processing time is a separate concern and must not overwrite occurrence time.

Future contracts may add provider timestamps or status-effective timestamps when needed.

## 10. Browser-side ingestion

Browser tracking is optimized for first-party, low-friction event emission.

Conceptual flow:

```text
Page / user action
      ↓
Trackfy browser SDK
      ↓
Create/reuse visitor + session context
      ↓
Capture acquisition metadata
      ↓
Assign event_id
      ↓
POST to Trackfy ingestion endpoint
      ↓
Acceptance / rejection response
```

Browser payloads must be treated as untrusted input.

The server must derive authoritative fields such as `received_at`, validate tenant/site credentials, enforce payload limits and reject malformed or unauthorized data.

Browser events may be delayed, duplicated, abandoned by navigation or arrive out of order. The pipeline must explicitly tolerate those conditions.

## 11. Server-side ingestion

Server-side ingestion is used for events that are more reliable, authoritative or inaccessible from the browser.

Conceptual flow:

```text
External provider / backend
          ↓
Authenticated API or webhook
          ↓
Integration event envelope
          ↓
Provider validation
          ↓
Canonical normalization
          ↓
Canonical event
```

For provider-driven events, Trackfy should retain both:

1. the normalized canonical event;
2. enough provider provenance to trace the canonical event back to the original external assertion.

## 12. Deduplication

Deduplication and idempotency are related but distinct concepts.

### Event-level deduplication

When the same `event_id` is delivered more than once, Trackfy must not create multiple canonical facts.

Expected behavior:

```text
first delivery  → accepted
same event_id   → duplicate / replay response
```

The duplicate must remain observable for debugging without becoming a second business event.

### Provider-level deduplication

Providers may not supply a Trackfy `event_id`. In that case, a provider-specific uniqueness key must be derived from stable provider identifiers under an explicit integration rule.

Never deduplicate solely by an unsafe heuristic such as `(event_name, timestamp, amount)` when a provider identifier is available.

## 13. Idempotency

Idempotency protects the effect of retrying a request. Deduplication protects the canonical event identity.

For APIs that create business outcomes, clients should provide an idempotency key where defined.

Rules:

- the same idempotency key with the same semantic request must return the same logical outcome;
- reuse of an idempotency key with materially different request content must be rejected;
- idempotency retention must be long enough to cover the documented retry window;
- idempotency records must be scoped to the tenant/integration boundary;
- an idempotency response must not imply that a duplicate request created a second order, refund or lifecycle transition.

The exact retention period belongs to the API reliability specification.

## 14. Source of truth

Trackfy uses an explicit hierarchy.

### Raw external assertion
The original provider event/request is evidence of what an external system reported.

### Canonical event
The normalized Trackfy event is the canonical tracking fact for internal processing.

### Derived state
Sessions, current order status, attribution results, aggregates, dashboards and alerts are derived representations.

### UI state
A dashboard or debugger screen is never the source of truth.

A provider may remain authoritative for a business fact such as payment approval, while Trackfy remains authoritative for the fact that it received and normalized that provider assertion. This distinction must be preserved.

## 15. Correlation rules

Correlation attempts should proceed from strongest available signals to weaker contextual signals.

Conceptually:

```text
Explicit Trackfy ID
  ↓
Provider/external ID + integration scope
  ↓
Click ID
  ↓
Session ID
  ↓
Visitor ID
  ↓
Other configured deterministic correlation
  ↓
Unmatched
```

Heuristic matching must never silently upgrade uncertain identity into fact. An uncertain association should remain explicitly marked as matched-with-confidence or unmatched once those concepts are introduced.

For v0.1, the safest default is deterministic matching only.

## 16. Order and payment events

Tracking events and commerce lifecycle facts must remain separable.

Example:

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

A status update must reference the relevant Order and preserve the external source that asserted the transition.

The system must not infer `approved` merely because `purchase` occurred unless the source contract explicitly defines that equivalence.

## 17. Processing outcomes

Every accepted ingestion should have a processing state that can be inspected later.

Initial conceptual states:

- `accepted`
- `duplicate`
- `rejected`
- `processed`
- `failed`
- `unmatched`

These are processing states, not business lifecycle states.

For example, an event can be `processed` while the referenced order remains `pending`.

## 18. Validation

Validation occurs in layers.

### Transport validation
Authentication, request size, content type, rate limits and basic syntax.

### Contract validation
Required fields, field types, enum/version compatibility and payload shape.

### Tenant validation
Organization, workspace, site and credential scope.

### Semantic validation
Event-specific invariants such as required order identifiers for `refund`.

### Correlation validation
Whether supplied visitor/session/click/order relationships are structurally valid.

A validation failure must produce a machine-readable reason and a debugger-visible trace where permitted.

## 19. Contract versioning

Every event type carries `event_version`.

Rules:

1. Existing versions remain readable for their supported retention window.
2. Breaking changes require a new version.
3. Producers should migrate explicitly rather than relying on silent server rewriting.
4. Derived processing must know which contract version produced the fact.
5. The canonical fact must retain its original version for replay.

The exact compatibility policy and deprecation window will be defined before public API launch.

## 20. Security and privacy boundary

Tracking payloads are untrusted and potentially privacy-sensitive.

The contract therefore requires:

- authenticated server-side integration paths;
- tenant-scoped credentials;
- no secret/service credentials in browser payloads;
- explicit payload size limits;
- safe logging that avoids leaking unnecessary personal data;
- source-aware retention/deletion controls;
- authorization before exposing event traces across workspace boundaries.

The contract does not yet define Trackfy's complete privacy/compliance policy. Those requirements must be resolved before production collection of regulated or sensitive data.

## 21. Event Debugger requirements derived from the contract

The Event Debugger must be able to show, for an event where the viewer is authorized:

```text
Event ID
Event type + version
Occurred at
Received at
Source / provider
Organization / Workspace / Site
Visitor / Session / Click IDs when present
UTM + provider click identifiers
Payload summary
Processing state
Deduplication result
Correlation result
Downstream references
Errors / rejection reason when applicable
```

The debugger should expose a timeline rather than only isolated records, so operators can inspect:

```text
Click → Session → Page View → Lead → Checkout → Order → Status Update
```

## 22. Acceptance criteria

### AC-01 — Immutable identity
Given an accepted event, its `event_id`, source evidence and occurrence timestamp remain stable across downstream processing.

### AC-02 — Duplicate delivery
Given two deliveries with the same canonical `event_id`, exactly one canonical event exists and the second delivery is observable as a duplicate/replay.

### AC-03 — Distinct timestamps
Given a late event, `occurred_at` preserves source occurrence time while `received_at` records Trackfy ingestion time.

### AC-04 — Acquisition preservation
Given a browser event containing UTM parameters and a provider click identifier, both are preserved independently in the canonical event.

### AC-05 — Missing identifiers
Given a valid event without a visitor or session identifier, Trackfy may accept it when the event contract permits the omission and must not invent a false identity.

### AC-06 — Tenant isolation
Given a valid event credential scoped to Workspace A, the event cannot be written into Workspace B.

### AC-07 — Provider provenance
Given a provider webhook containing an external event/order ID, the canonical event remains traceable to the originating integration/provider context.

### AC-08 — Replayability
Given canonical event evidence and the same deterministic processing configuration, derived state can be recomputed without relying on dashboard state.

### AC-09 — Lifecycle separation
Given an order receives a provider status update, the event records the asserted transition without erasing the previous lifecycle history.

### AC-10 — Untrusted browser
Given a browser event containing forged server-authoritative fields such as `received_at`, the server ignores or replaces those fields with authoritative values.

## 23. Explicit non-goals for v0.1

This contract does not yet finalize:

- public API endpoint paths;
- authentication protocol details;
- JSON Schema/OpenAPI syntax;
- SDK implementation;
- exact event retention periods;
- consent management model;
- exact session timeout;
- identity stitching algorithms beyond deterministic correlation;
- attribution calculations;
- financial calculation rules;
- database schema;
- queue/event-bus technology;
- analytics storage technology.

Those belong to later specifications and architecture decisions.

## 24. Next contract work

The next specification should define the **Attribution Model v0.1** against these canonical facts, including touchpoint eligibility, attribution windows, first-touch, last-touch and multi-touch behavior, and how attribution interacts with approved revenue, cancellations, refunds, fees, commissions, ad spend and profit.
