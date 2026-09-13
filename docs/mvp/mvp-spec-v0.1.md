# Trackfy — MVP Spec v0.1

**Status:** Draft  
**Version:** 0.1  
**Scope:** First release that validates Trackfy's core value proposition: reliable tracking, conversion lifecycle visibility, deterministic attribution, revenue/cost/profit analytics and operational debugging in a multi-tenant SaaS.

This document turns the Product Spec, Domain Model, Event/Tracking Contracts and Attribution Model into an intentionally constrained implementation target. It does not define final architecture, SQL schema, UI design system or every future integration.

## 1. MVP thesis

The MVP is successful if a user can connect one tracked site and at least one conversion source, receive trustworthy acquisition and conversion events, see the complete journey from traffic to sale, understand attribution, and reconcile revenue, costs and profit without needing to inspect raw technical systems.

The MVP must demonstrate that Trackfy is:

- more understandable than a conventional tracking dashboard;
- multi-tenant by construction;
- reliable across browser and server-side event delivery;
- explicit about uncertainty instead of inventing attribution;
- operationally debuggable;
- useful for performance decisions, not merely event collection.

## 2. MVP user outcome

A Media Buyer or Analyst should be able to answer from Trackfy:

1. Which campaign/source generated the traffic?
2. Which visitors/sessions/clicks progressed to lead, checkout and purchase?
3. Which Orders are pending, approved, rejected, cancelled, refunded or otherwise adjusted?
4. How much gross, approved, pending and net revenue was generated?
5. How much was spent on advertising and other configured costs?
6. What is the resulting ROAS and profit?
7. Which touchpoint received attribution credit and why?
8. Where did tracking fail, duplicate, become unmatched or diverge between browser and server?

## 3. MVP scope

### 3.1 Tenant and workspace foundation — MUST

- User authentication boundary.
- Organization creation.
- Workspace creation.
- Organization membership.
- Initial RBAC roles required by MVP:
  - Owner
  - Admin
  - Analyst
  - Media Buyer
  - Viewer
- Site creation and site-scoped tracking configuration.
- Tenant-safe API authorization.
- Site/publishable tracking credential separation from server secrets.

### 3.2 Tracking collection — MUST

- Trackfy Browser SDK or equivalent first-party browser collector.
- First-party/custom tracking endpoint.
- Canonical event envelope v0.2.
- Standard events:
  - `page_view`
  - `landing_view`
  - `click`
  - `lead`
  - `checkout_started`
  - `checkout_completed`
  - `order_created`
  - `purchase`
  - `order_status_changed`
  - `refund`
  - `chargeback`
  - `tracking_error`
- Custom events using the canonical envelope.
- UTM preservation.
- `fbclid`, `gclid`, `ttclid` support.
- Internal `visitor_id`, `session_id` and `click_id`.
- `event_id` generation/preservation.
- `occurred_at` and authoritative `received_at`.
- Server-side ingestion API.
- Authenticated webhook ingestion for at least one sales/conversion source or generic webhook contract.
- Deduplication.
- Idempotent mutation handling.
- Out-of-order and late-arriving event tolerance.

### 3.3 GTM compatibility — MUST at launch target

Trackfy must provide a supported integration path for Google Tag Manager Web.

MVP target:

```text
Website → GTM Web → Trackfy
```

The integration must document:

- Data Layer event mapping;
- required variables;
- event ID propagation/generation;
- site identification;
- consent handling boundary;
- validation/debugging behavior.

Full first-class GTM Server template/package automation may be delivered immediately after the first MVP cut if it does not block the canonical ingestion path. The contract must not prevent it.

### 3.4 Conversion and commerce lifecycle — MUST

Trackfy must represent:

```text
pending
approved
rejected
cancelled
refunded
partially_refunded
chargeback
```

The MVP must retain lifecycle transition evidence and distinguish current projected status from historical assertions.

An Order must support, at minimum:

- internal order identity;
- external order identity + provider scope;
- currency;
- gross amount;
- lifecycle status;
- source/provider;
- timestamps;
- correlation references;
- financial adjustments relevant to reporting.

### 3.5 Attribution — MUST

Initial models:

- First Touch;
- Last Touch;
- Linear Multi-touch.

MVP must support:

- configurable attribution window;
- eligible touchpoint filtering;
- deterministic attribution;
- unattributed outcome;
- attribution model/version recording;
- explanation of eligible touchpoints and credit allocation;
- recomputation when canonical facts or configuration change.

MVP does not require arbitrary user-defined attribution formulas.

### 3.6 Revenue, costs and profitability — MUST

MVP reporting must distinguish at least:

- gross revenue;
- approved revenue;
- pending revenue;
- refunds;
- net revenue;
- fees;
- commissions;
- product/direct costs where configured;
- ad spend;
- net profit;
- ROAS.

The MVP must never represent profit as equivalent to revenue.

Initial conceptual formulas:

`Net Revenue = Gross Revenue - Refunds - applicable adjustments`

`Net Profit = Net Revenue - Fees - Commissions - Direct Costs - Ad Spend`

Exact accounting/tax semantics are outside MVP scope.

### 3.7 Dashboard — MUST

The initial dashboard must answer operational questions without requiring raw event inspection.

Minimum views:

1. Overview
2. Campaign/source performance
3. Sales/order lifecycle
4. Revenue/cost/profit
5. Funnel
6. Event Debugger
7. Tracking Health

Core metrics should include:

- visitors;
- sessions;
- clicks;
- leads;
- checkouts;
- orders;
- approved orders;
- revenue;
- ad spend;
- profit;
- ROAS;
- conversion rates;
- unmatched/duplicate events.

### 3.8 Event Debugger — MUST

For an authorized trace, the MVP must show:

```text
Click
 ↓
Session
 ↓
Page View
 ↓
Lead
 ↓
Checkout
 ↓
Order
 ↓
Status / Refund
```

Each event should expose its ID, type, time, source, identity references, acquisition metadata, processing state and errors when present.

The debugger must distinguish:

- received;
- duplicate;
- rejected;
- processed;
- unmatched;
- downstream delivery failure.

### 3.9 Tracking Health — MUST

The MVP should calculate actionable health signals for:

- event volume anomalies;
- duplicate rate;
- unmatched rate;
- missing key identifiers;
- browser/server divergence where both paths exist;
- ingestion latency;
- integration delivery failures.

The health score is a derived diagnostic, never a canonical source fact.

## 4. MVP integration strategy

The MVP should avoid attempting every advertising platform simultaneously.

The integration abstraction must be provider-neutral, while launch integrations should be selected according to user value and operational feasibility.

Minimum integration capability:

```text
Inbound:
Browser / GTM / S2S / Webhook

Outbound target abstraction:
Provider adapter + generic webhook
```

A generic outbound webhook is required so the routing model is testable before a large catalog of provider adapters exists.

Priority provider integrations can then be added without changing the canonical contract.

## 5. MVP end-to-end flow

### Flow A — acquisition

```text
Ad
 ↓
Landing URL with UTM/click ID
 ↓
Trackfy collector
 ↓
Visitor + Session + Click
 ↓
page_view / click events
```

### Flow B — conversion

```text
Visitor
 ↓
lead
 ↓
checkout_started
 ↓
order_created
 ↓
purchase
 ↓
pending
 ↓
approved
```

### Flow C — correction

```text
approved
 ↓
partial_refund
 ↓
refunded
```

Derived revenue/profit changes, while historical evidence remains intact.

### Flow D — debugging

```text
Operator searches Order
 ↓
opens Event Trace
 ↓
sees acquisition path
 ↓
sees missing/duplicate/mismatched events
 ↓
sees attribution explanation
 ↓
sees downstream routing status
```

## 6. MVP non-goals

The following are explicitly outside the first MVP unless required to validate the core thesis:

- full CRM;
- customer support suite;
- product catalog management;
- checkout hosting;
- payment processing;
- ad buying/optimization;
- arbitrary BI replacement;
- advanced cohort/LTV modeling;
- predictive ML attribution;
- AI-generated campaign optimization;
- dozens of provider-specific adapters;
- enterprise SSO/SCIM;
- complex billing automation;
- custom workflow engine;
- unrestricted custom attribution formulas.

These may become later product capabilities but must not distort the first implementation.

## 7. MVP acceptance criteria

### AC-01 — Tenant isolation
A user from Organization A cannot read or mutate resources belonging to Organization B through any API, worker or dashboard query.

### AC-02 — Browser event ingestion
A valid browser event is accepted, receives authoritative ingestion metadata and becomes visible in the event trace.

### AC-03 — Duplicate safety
Sending the same `event_id` twice produces one canonical event and an observable duplicate outcome.

### AC-04 — Server conversion
A server/webhook conversion can create or update an Order without requiring a browser event to be present.

### AC-05 — Lifecycle visibility
An Order changing from pending to approved is represented as a historical lifecycle transition and a current derived state.

### AC-06 — Attribution reproducibility
Running the same attribution model/version/configuration over the same canonical facts produces the same credit allocation.

### AC-07 — No fabricated attribution
An Order without eligible evidence remains unattributed rather than being assigned to an arbitrary source.

### AC-08 — Financial correction
A refund reduces the appropriate derived revenue/profit metrics without deleting or rewriting the original purchase evidence.

### AC-09 — Debugger trace
An authorized user can inspect a conversion journey from acquisition through Order lifecycle and see processing failures or missing correlations where applicable.

### AC-10 — Routing isolation
A downstream provider failure does not delete, duplicate or invalidate the canonical event or Order.

### AC-11 — GTM path
A supported GTM Web configuration can emit Trackfy events with the required site and event identity fields.

### AC-12 — Health visibility
The system exposes actionable tracking-quality signals when duplicate, unmatched, delayed or failing events are detected.

## 8. MVP quality bar

MVP is not considered complete merely because the happy path works.

The release must also demonstrate:

- tenant isolation;
- duplicate/retry safety;
- deterministic attribution;
- lifecycle correction behavior;
- traceability from source evidence to dashboard metric;
- observable asynchronous failures;
- reproducible analytics results;
- safe handling of malformed/untrusted browser input.

## 9. MVP implementation order

Implementation should follow the dependency graph rather than UI-first development:

```text
Contracts
  ↓
Tenant/Auth boundary
  ↓
Event ingestion
  ↓
Canonical event persistence
  ↓
Identity/correlation
  ↓
Order lifecycle
  ↓
Attribution
  ↓
Financial projections
  ↓
Analytics read models
  ↓
Debugger / Health
  ↓
Dashboard
  ↓
Provider adapters
```

Every phase must have executable tests before moving downstream.

## 10. Definition of Done

The MVP is ready for controlled release only when:

1. All MUST capabilities above are implemented or explicitly waived by a recorded decision.
2. Critical acceptance criteria are covered by automated tests.
3. Tenant isolation has negative authorization tests.
4. Duplicate/idempotency behavior has retry tests.
5. Event replay/recalculation produces deterministic derived results.
6. Financial lifecycle corrections are verified against known scenarios.
7. GTM ingestion has a documented working fixture.
8. The Event Debugger can explain a complete successful and failed journey.
9. Tracking Health detects seeded failure scenarios.
10. The system has enough observability to diagnose ingestion, processing and delivery failures.

## 11. Explicit deferred decisions

The MVP intentionally leaves these for subsequent engineering decisions:

- exact provider launch list;
- exact message broker/queue;
- exact analytical database;
- event retention duration;
- production scale/SLO targets;
- precise session timeout;
- exact first-party identity storage mechanism;
- advanced cross-domain identity transfer;
- privacy/compliance policy details;
- billing plans and entitlements.

## 12. Next step

The next engineering artifact should be the implementation harness: repository governance, specification workflow, testing strategy, contract-test conventions, evaluation fixtures, CI gates and coding-agent operating rules.
