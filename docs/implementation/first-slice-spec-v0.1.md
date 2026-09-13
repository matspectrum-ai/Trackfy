# Trackfy — First Implementation Slice Spec v0.1

**Status:** Draft
**Version:** 0.1
**Purpose:** Define the first production-shaped vertical slice of Trackfy before implementation begins.

## 1. Objective

Implement one complete, verifiable path through the Trackfy system without attempting to build the whole product.

The slice must prove the foundational architecture:

```text
Browser / HTTP producer
        ↓
Tracking ingestion
        ↓
Tenant + Site validation
        ↓
Canonical Event
        ↓
Durable persistence
        ↓
Derived processing
        ↓
Event trace
        ↓
Dashboard / Debugger
```

The slice is successful only when a real event can enter the system, remain tenant-isolated, be persisted as canonical evidence, be inspected through the application, and be tested end-to-end.

## 2. Scope

### MUST implement

1. Local development environment.
2. Go tracking/API service.
3. PostgreSQL persistence.
4. Redis-backed asynchronous boundary where required by the slice.
5. Next.js dashboard shell.
6. Better Auth authentication foundation.
7. Organization / Workspace / Site tenant hierarchy sufficient for the slice.
8. A publishable Site tracking identifier.
9. Browser SDK minimum producer.
10. Canonical event ingestion endpoint.
11. `page_view` and `click` events.
12. Event IDs, occurrence/receipt timestamps and acquisition metadata.
13. Event deduplication by canonical event identity.
14. Tenant/site authorization at ingestion.
15. Canonical event persistence.
16. Basic event processing state.
17. Event trace read model.
18. Event Debugger page.
19. Basic dashboard event counters.
20. Automated unit, contract and end-to-end tests.
21. Deterministic local verification.

### MUST NOT implement yet

- full attribution engine;
- financial profitability engine;
- all ad-platform adapters;
- production GTM templates;
- complex identity recovery;
- custom attribution models;
- billing;
- advanced alerts/rules;
- ML-based attribution;
- full analytics warehouse optimization;
- Kubernetes;
- Kafka/Pulsar/NATS unless a measured requirement appears;
- premature microservice decomposition.

The slice must prove the contracts without pretending to be the complete SaaS.

## 3. Primary user journey

The first complete journey is:

```text
User signs in
   ↓
Creates/enters Organization
   ↓
Creates Workspace
   ↓
Creates Site
   ↓
Copies Site tracking snippet/config
   ↓
Site emits page_view
   ↓
Trackfy accepts event
   ↓
Event appears in Debugger
   ↓
Click event is emitted
   ↓
Click appears in same trace
```

A second verification path must prove rejection/isolation:

```text
Site A credential
   ↓
attempt to submit against Site B
   ↓
rejected
   ↓
no canonical event created for Site B
```

## 4. First-slice domain subset

```text
User
  ↓
Organization Membership
  ↓
Organization
  ↓
Workspace
  ↓
Site
  ↓
Visitor
  ↓
Session
  ↓
Event
```

`Click` is represented as an event plus explicit click identity where required by the tracking contract.

Orders, Customers, Leads, Checkout, Attribution and financial entities remain modeled but are not implemented in this slice.

## 5. Ingestion contract

### Endpoint intent

The service must expose a public-site-safe ingestion path for browser events and an authenticated server ingestion path for trusted producers.

Conceptual endpoints:

```text
POST /v1/events
POST /v1/events/batch
POST /v1/ingest/webhook/:integration
```

Exact routing names are implementation details unless promoted into the public API contract.

### Browser producer constraints

The browser may send:

- site tracking identifier;
- event ID;
- event name;
- occurrence timestamp;
- visitor/session identifiers when available;
- acquisition metadata;
- page context;
- event payload.

The server derives or verifies:

- received timestamp;
- site scope;
- authorization scope;
- processing state;
- canonical persistence identity.

No secret provider credential may be present in the browser producer.

## 6. Canonical events in the slice

### `page_view`

Minimum semantic data:

```json
{
  "event_name": "page_view",
  "payload": {
    "url": "https://example.com/landing",
    "path": "/landing"
  }
}
```

### `click`

Minimum semantic data:

```json
{
  "event_name": "click",
  "payload": {
    "target": "cta",
    "href": "https://example.com/checkout"
  }
}
```

Both use the v0.2 canonical envelope. No alternative event envelope may be invented for this slice.

## 7. Identity requirements

The implementation must support:

- `event_id`;
- `visitor_id`;
- `session_id`;
- `click_id` for click events where applicable;
- organization/workspace/site scope.

The slice does not need full cross-device identity or advanced probabilistic matching.

Identity relationships must be explicit and tenant-scoped.

## 8. Acquisition requirements

The browser producer must preserve, when present:

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
```

The raw received values must remain inspectable in the Debugger.

## 9. Deduplication requirements

Given two submissions with the same `event_id` under the same tenant/event identity scope:

```text
first → one canonical event
second → duplicate processing outcome
```

The duplicate must not create a second canonical event.

The system must expose enough metadata to explain that the second delivery was a duplicate.

## 10. Persistence boundary

The slice uses PostgreSQL as the authoritative transactional store.

Minimum persisted concepts:

```text
users/auth identities
organizations
memberships
workspaces
sites
site tracking credentials/identifiers
canonical events
event processing records
```

Physical schema names are implementation decisions after this slice is accepted.

The implementation must keep source evidence distinguishable from derived processing state.

## 11. Async processing

The slice must demonstrate at least one asynchronous processing boundary if the selected architecture requires it.

Candidate flow:

```text
ingestion
   ↓
persist canonical event
   ↓
queue
   ↓
processor
   ↓
processing/read-model update
```

The synchronous acknowledgment must not depend on dashboard analytics computation.

Redis is the initial candidate for this boundary.

## 12. Event Debugger

The Debugger must allow an authorized user to inspect an event trace.

Minimum display:

```text
Event ID
Event type
Event version
Occurred at
Received at
Source
Organization
Workspace
Site
Visitor ID
Session ID
Click ID when present
UTM values when present
Provider click IDs when present
Payload summary
Processing state
Duplicate status
```

The trace view must visually establish ordering:

```text
page_view → click
```

No debugger action may mutate canonical event evidence.

## 13. Dashboard

The first dashboard is intentionally small.

It must show at minimum:

- total events for selected site/workspace;
- page views;
- clicks;
- recent event activity;
- processing failures;
- navigation to Event Debugger.

Numbers shown by the dashboard must be derived from persisted facts, not hard-coded demo state.

## 14. Authentication and authorization

The slice must support:

- sign-in;
- authenticated application session;
- Organization membership;
- Workspace scope;
- Site scope;
- authorization check on Debugger reads;
- authorization check on administrative operations.

A valid user from Organization A must not read Organization B's event data.

A Site tracking credential identifies ingestion scope; it does not grant dashboard privileges.

## 15. Security requirements

The slice must enforce:

1. Site ingestion credentials are not privileged application credentials.
2. Browser events are untrusted input.
3. Request limits are enforced.
4. Payload size is bounded.
5. Tenant authorization occurs before protected reads/writes.
6. Sensitive values are not emitted into ordinary application logs.
7. SQL access is parameterized through the chosen data-access layer.
8. Secrets are supplied through environment/secret management, never committed.
9. Cross-tenant test coverage exists.

## 16. API error contract

Failures must be machine-readable and stable enough for automated tests.

Conceptual categories:

```text
invalid_request
unauthorized
forbidden
invalid_site
invalid_event
duplicate_event
rate_limited
internal_error
```

Exact HTTP mapping is finalized during API contract implementation.

## 17. Test strategy

### Unit tests

Cover:

- envelope validation;
- event ID validation;
- acquisition normalization;
- tenant scope validation;
- deduplication decisions;
- processing-state transitions;
- authorization decisions.

### Contract tests

Verify browser and server producers conform to the canonical event contract.

### Integration tests

Verify:

```text
API → PostgreSQL
API → Redis/processor
Auth → authorization boundary
Debugger → persisted event trace
```

### End-to-end tests

At minimum:

1. sign in;
2. create/select tenant context;
3. create/select site;
4. send `page_view`;
5. observe it in Debugger;
6. send `click`;
7. observe same trace;
8. resend same `event_id`;
9. verify no duplicate canonical event;
10. attempt cross-tenant read/ingestion;
11. verify rejection.

## 18. Acceptance criteria

### AC-01 — Real browser event
A minimal Trackfy browser producer can emit a valid `page_view` and receive a successful ingestion response.

### AC-02 — Canonical persistence
The accepted event is persisted as one canonical fact with stable `event_id` and both occurrence and receipt timestamps.

### AC-03 — Debugger visibility
An authorized operator can locate and inspect the event through the dashboard Debugger.

### AC-04 — Click trace
A `click` event for the same visitor/session context appears as a subsequent trace item.

### AC-05 — Duplicate safety
A repeated submission with the same canonical `event_id` does not create a second canonical event.

### AC-06 — Tenant isolation
Credentials/data from one tenant cannot read or create data in another tenant's scope.

### AC-07 — Raw acquisition preservation
UTM/click-ID values sent by the producer remain inspectable without silent mutation.

### AC-08 — Failure visibility
Invalid events expose a machine-readable error and are distinguishable in operational/debugging views where authorized.

### AC-09 — Deterministic tests
The full first-slice test suite passes repeatedly from a clean local environment.

### AC-10 — No dashboard source of truth
Removing/rebuilding the dashboard read model does not alter canonical event evidence.

## 19. Verification commands

The repository must provide documented commands equivalent to:

```text
install
lint
unit test
contract test
integration test
e2e test
build
```

The exact package manager/runtime commands are finalized with the repository bootstrap.

## 20. Implementation order

Work must proceed in this order:

```text
1. Repository/toolchain bootstrap
2. Database + migration foundation
3. Authentication foundation
4. Tenant/RBAC foundation
5. Site registration + tracking credential
6. Canonical event model
7. Ingestion endpoint
8. Deduplication
9. Async processing boundary
10. Debugger read model/API
11. Dashboard shell + event views
12. Browser SDK
13. End-to-end verification
14. Hardening / observability
```

Each step must have tests before the next dependent step is considered complete.

## 21. Definition of Done

The slice is complete only when:

- acceptance criteria AC-01 through AC-10 pass;
- unit/contract/integration/E2E tests are green;
- no cross-tenant access path is demonstrated;
- canonical events survive derived-state rebuild;
- duplicate delivery is safe;
- browser and server ingestion boundaries are explicit;
- no provider-specific schema has leaked into the canonical model;
- no production secret is present in repository contents;
- local setup is reproducible from a clean checkout;
- relevant changes are documented;
- CI executes the required verification commands;
- the final commit is small enough to review and describes the completed slice accurately.

## 22. Explicit non-goals

This specification deliberately does not require the first slice to prove:

- millions of events per second;
- every advertising integration;
- final attribution correctness for all business cases;
- final analytics performance at scale;
- full production privacy/compliance posture;
- global multi-region deployment;
- mobile SDKs;
- probabilistic identity resolution.

Those are later engineering slices and must be introduced through their own contracts and acceptance criteria.

## 23. Next engineering artifact

Before implementation begins, the repository should define the concrete machine-readable contracts for:

1. API request/response schemas.
2. Canonical event JSON Schema.
3. Database migration contract.
4. Auth/session contract.
5. Site tracking credential contract.
6. Browser SDK public API.
7. Debugger query/read-model contract.
8. Test fixtures and golden event traces.

Those artifacts convert this implementation slice from prose into executable engineering constraints.
