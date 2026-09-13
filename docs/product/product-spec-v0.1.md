# Trackfy — Product Specification v0.1

**Status:** Draft
**Version:** 0.1
**Product:** Trackfy
**Positioning:** Tracking, Attribution & Revenue Intelligence

## 1. Product definition

Trackfy is a multi-tenant SaaS for collecting marketing and business events, connecting those events into a measurable customer journey, attributing conversions and revenue to acquisition sources, and turning performance data into understandable operational decisions.

The product is not centered on UTM parameters. UTMs are one identity and acquisition signal among several. The core abstraction is an observable event graph that can connect acquisition, behavior, conversion and financial outcomes.

The product goal is to answer, with traceable evidence:

> Which acquisition source generated this result, what happened along the journey, how much revenue did it produce, what did it cost, and what was the resulting profit?

## 2. Product thesis

Trackfy should use UTMify as a functional baseline, not as an implementation blueprint or a product identity. The target is a platform that is simpler to operate, easier to understand, strongly multi-user, and more transparent about tracking health and attribution.

The primary product differentiators are:

1. A unified event graph from acquisition to financial outcome.
2. Observable tracking with a first-class Event Debugger.
3. Clear separation between facts/events and derived analytics.
4. Revenue intelligence that distinguishes gross revenue, approved revenue, pending revenue, refunds, fees, costs, ad spend and profit.
5. Multi-tenant collaboration designed into the product rather than added later.
6. A simpler onboarding and operating model despite broad integrations.

The source material explicitly proposes the positioning around discovering which ad generated each result “from click to profit”, and identifies observability, server-side tracking, revenue lifecycle and multi-user support as central product concerns. fileciteturn3file3L1-L5 fileciteturn3file5L1-L17

## 3. Problem

Marketing teams often have data distributed across ad platforms, landing pages, trackers, checkout systems, sales platforms and payment lifecycle events. The existence of a sale alone is insufficient: teams need to connect the sale to the acquisition path and understand its financial result.

The operational problems Trackfy targets are:

- attribution data that is incomplete, opaque or difficult to validate;
- tracking failures that are discovered only after revenue is lost;
- fragmented lifecycle states for orders and payments;
- dashboards that report revenue without adequately explaining costs and profit;
- UTM-centric models that fail to represent the full customer journey;
- configuration-heavy tools that become difficult to operate as integrations grow;
- weak support for organizations, workspaces, members and role-based access;
- difficulty determining whether a conversion was received, matched, attributed and financially reconciled.

The source material specifically frames the Event Debugger as a way to locate where a missing sale broke in the pipeline, for example separating click/session/checkout/purchase/postback success or failure. fileciteturn3file3L1-L11

## 4. Product goals

### G1 — Reliable measurement
Capture and preserve the events necessary to reconstruct customer journeys across browser-side and server-side sources.

### G2 — Traceable attribution
Connect acquisition signals to conversions and revenue through explicit attribution rules rather than opaque aggregation.

### G3 — Financial clarity
Expose gross revenue, approved revenue, pending revenue, refunds, fees, commissions, costs, ad spend and profit as distinct concepts.

### G4 — Operational observability
Allow an operator to inspect the lifecycle of a tracked conversion and identify where data was missing, rejected, duplicated, delayed or unmatched.

### G5 — Simple operation
Provide a small number of understandable setup steps and sensible defaults so a user can become operational without understanding the internal architecture.

The proposed onboarding in the source material is intentionally simple: connect a sales source, connect an ad account, then install the tracker. fileciteturn3file0L1-L8

### G6 — Multi-user SaaS by design
Support organizations, workspaces, members, roles and scoped resources from the beginning.

### G7 — Extensible integration surface
Support ad platforms, sales platforms, webhooks and external event sources without coupling the product core to any one provider.

## 5. Non-goals for Product Spec v0.1

The following are intentionally not fully specified here:

- exact database tables and indexes;
- exact event JSON schemas;
- exact attribution mathematics and tie-breaking rules;
- exact provider-by-provider API behavior;
- exact infrastructure technology choices beyond architectural principles;
- detailed pricing and packaging;
- AI recommendation behavior;
- implementation language, framework and repository layout;
- final MVP boundary.

These topics belong to later specifications and must not be silently inferred from this document.

## 6. Target personas

### 6.1 Performance marketer / media buyer
Needs to compare campaigns, sources and acquisition performance; understand spend, conversions, revenue, ROAS and profit; and quickly identify underperforming traffic.

### 6.2 Affiliate / seller / operator
Needs reliable attribution across a sales journey and a clear view of orders, approvals, refunds, fees and profit.

### 6.3 Agency operator
Manages multiple clients or business units and needs organizations/workspaces, member access, role boundaries, reusable integrations and isolated reporting contexts.

### 6.4 Analyst
Needs trustworthy event and financial data, flexible filtering and traceability from aggregate metrics back to underlying lifecycle events.

### 6.5 Developer / integration operator
Needs APIs, API keys, webhooks, event ingestion, debugging information and clear contracts for external systems.

### 6.6 Viewer / executive
Needs concise dashboards that answer what happened and whether performance is healthy without exposing unnecessary configuration complexity.

## 7. Core product concepts

Trackfy's product model is centered on the following conceptual chain:

`Organization → Workspace → Site → Visitor → Session → Click → Event → Lead → Checkout → Order → Payment Lifecycle → Attribution`

This is a conceptual ordering, not yet a final persistence model.

### 7.1 Acquisition identity
Trackfy must be capable of retaining and resolving signals such as:

- `utm_source`
- `utm_medium`
- `utm_campaign`
- `utm_content`
- `utm_term`
- `fbclid`
- `gclid`
- `ttclid`
- `click_id`
- `session_id`
- `visitor_id`
- `order_id`
- `customer_id`

The source material explicitly treats these as complementary identity sources rather than making UTM the sole tracking primitive. fileciteturn3file6L1-L18

### 7.2 Event graph
The system should be able to represent journeys such as:

`Ad → Click → Session → Landing → Lead → Checkout → Purchase → Approval → Refund`

A sale is therefore not treated as an isolated fact; it is part of a sequence of related events. fileciteturn3file6L14-L18

### 7.3 Revenue lifecycle
An order/conversion may move through distinct states such as:

`pending → approved`

and may also become:

`rejected`, `cancelled`, `refunded`, `chargeback`, `partially_refunded`.

The source material requires these states to remain distinguishable rather than collapsing everything into a single “sale” metric. fileciteturn3file1L1-L18

## 8. Core product modules

### 8.1 Dashboard
Primary performance overview. Must make the relationship among traffic, conversions, revenue, costs and profit understandable.

Representative metrics include:

- clicks;
- sessions;
- leads;
- checkouts;
- purchases;
- approved sales;
- pending sales;
- gross revenue;
- approved revenue;
- refunds;
- fees and commissions;
- ad spend;
- profit;
- ROAS.

The example dashboard in the source material explicitly combines clicks, leads, sales, revenue, profit and ROAS. fileciteturn3file0L9-L16

### 8.2 Tracking
Collect first-party/browser-side and server-side events, preserve acquisition identifiers, associate events with visitors and sessions, and provide the basis for downstream attribution.

The source material proposes a flow through a tracking edge, event collector and event bus rather than relying only on browser JavaScript. fileciteturn3file3L12-L20

### 8.3 Event Debugger
Live or near-real-time event timeline for inspecting a journey and its processing state. The debugger is a first-class product feature, not an internal developer-only tool.

A representative timeline is:

`CLICK → PAGE_VIEW → LEAD → CHECKOUT → PURCHASE → POSTBACK → SALE_UPDATED`

The debugger should make success, failure, absence, duplication and status transitions visible.

### 8.4 Attribution
Assign conversion and financial outcomes to acquisition interactions according to a selectable attribution model. The precise models and formulas will be defined in the Attribution Model specification.

Initial candidate families:

- first-touch;
- last-touch;
- linear/multi-touch;
- future custom models.

### 8.5 Revenue intelligence
Maintain separate financial concepts and derive operational metrics from them. The proposed financial relationship is:

`Gross Revenue - Refunds - Fees - Commissions - Product Cost = Net Revenue`

`Net Revenue - Ad Spend = Net Profit`

This relationship comes directly from the source material and will be formalized later with accounting semantics, rounding, currency and lifecycle rules. fileciteturn3file1L13-L24

### 8.6 Campaigns and acquisition
Provide a coherent representation of traffic sources, campaigns and acquisition identifiers so reporting does not require manually reconstructing naming conventions.

### 8.7 Integrations
Connect advertising platforms, sales/conversion platforms and external systems. The integration layer must normalize external data into Trackfy's domain without making provider-specific schemas the core model.

The source material names Meta, Google, TikTok, Hotmart and Kiwify among examples and emphasizes broad integration coverage. fileciteturn3file6L23-L27

### 8.8 Webhooks and API
Provide programmatic ingestion and outbound notification surfaces for external systems.

### 8.9 Organizations, workspaces and members
Provide multi-tenant collaboration with resource scoping and RBAC. The source material proposes Organization → Members → Workspaces → Projects/API Keys and roles including Owner, Admin, Analyst, Media Buyer, Viewer and Developer. fileciteturn3file0L1-L4

### 8.10 Rules and alerts
Provide a later automation surface for threshold-based or schedule-based actions, alerts and operational signals. Exact behavior is deferred.

### 8.11 Reports / cohorts / funnels
Provide analytical views that reuse the same underlying event and financial model rather than introducing separate, incompatible data representations.

## 9. Primary user journeys

### Journey A — Initial setup

1. User creates or joins an Organization.
2. User selects or creates a Workspace.
3. User connects a sales/conversion source.
4. User connects an advertising source.
5. User installs/configures tracking.
6. Trackfy receives the first event.
7. Trackfy validates the pipeline and communicates whether tracking is functioning.

Expected outcome: the user reaches a measurable state with minimum configuration.

### Journey B — Track a visitor

1. A user lands on a tracked site.
2. Acquisition parameters and identifiers are captured when present.
3. Trackfy establishes or resolves a Visitor.
4. Trackfy establishes or resolves a Session.
5. Interactions generate events.
6. Events are persisted as observable facts.

### Journey C — Track a conversion

1. A visitor enters the journey.
2. The visitor reaches a lead, checkout or purchase state.
3. A conversion/order event arrives.
4. Trackfy matches the conversion to available identity and journey data.
5. The order enters its lifecycle state.
6. Attribution is calculated according to the active model.
7. Revenue and cost metrics are updated from source facts.

### Journey D — Diagnose a missing sale

1. Operator opens Event Debugger.
2. Searches by relevant identifier such as order, click, visitor or session.
3. Inspects the event chain.
4. Sees which expected event exists, is missing, failed validation, was duplicated, or was not matched.
5. Uses the evidence to locate the integration or tracking failure.

This directly implements the observability concept established in the source material. fileciteturn3file3L1-L11

### Journey E — Evaluate campaign profitability

1. Operator selects a period and reporting scope.
2. Trackfy combines acquisition spend with conversions and financial lifecycle data.
3. Dashboard presents gross/approved/pending/refunded outcomes and costs separately.
4. Profit and ROAS are derived from defined rules.
5. Operator can drill down from aggregate metric to campaigns and underlying conversion evidence.

## 10. Functional requirements

### FR-001 — Tenant isolation
Every tenant-owned resource must have an unambiguous scope and must not be readable or mutable across unauthorized Organization/Workspace boundaries.

### FR-002 — Event observability
Every accepted event must retain enough metadata to determine its origin, identity, timing and processing state.

### FR-003 — Event immutability
Raw source events are treated as immutable facts. Corrections and state changes must be represented as additional facts or explicitly derived state, not by silently rewriting historical evidence.

### FR-004 — Identity resolution
The system must correlate available identifiers across visitor, session, click, event and order boundaries without assuming that one identifier is always present.

### FR-005 — Conversion lifecycle
The system must represent distinct order/payment states and preserve state transitions.

### FR-006 — Attribution traceability
An attributed result must be explainable from the source interactions and the active attribution model.

### FR-007 — Financial separation
The system must keep revenue, costs, fees, commissions, refunds and ad spend conceptually distinct before deriving profit metrics.

### FR-008 — Drill-down
Aggregate dashboard metrics must be traceable to the underlying dimension and lifecycle data used to calculate them.

### FR-009 — Debugging
An operator must be able to investigate an individual tracked journey without requiring direct database access.

### FR-010 — Multi-user access control
Resource access must depend on organization/workspace membership and role permissions.

### FR-011 — External ingestion
The system must support server-to-server/webhook ingestion in addition to browser-originated tracking.

### FR-012 — Idempotent processing readiness
The event and conversion model must permit duplicate delivery and retry without creating duplicate business outcomes. Exact idempotency contracts are defined later.

### FR-013 — Recomputable analytics
Derived metrics should be recomputable from canonical source facts and deterministic rules where practical.

### FR-014 — Integration normalization
External provider payloads must be mapped into canonical domain concepts while retaining provider provenance.

### FR-015 — Operational health
The product must expose tracking/integration health signals sufficient to tell an operator whether the measurement pipeline is functioning.

## 11. Acceptance criteria for Product Spec v0.1

The product specification is considered internally complete for the next design stage when:

- the product purpose and target users are unambiguous;
- the primary customer journey from acquisition to profit is defined;
- the event graph concept is explicit;
- conversion/payment lifecycle states are explicit;
- revenue and profit concepts are separated;
- the major product modules are identified;
- multi-tenant collaboration is a first-class requirement;
- the Event Debugger is a first-class product requirement;
- product boundaries and deferred decisions are explicitly recorded;
- no implementation detail is being treated as a product requirement unless it is necessary to satisfy the behavior.

## 12. Success signals

Product success will ultimately be measured using signals such as:

- time from account creation to first verified tracked event;
- percentage of connected conversion flows that can be traced end-to-end;
- percentage of conversions that can be attributed under the active model;
- time required to diagnose a missing/misrouted conversion;
- consistency between source-system and Trackfy financial lifecycle totals;
- active organizations and collaborative usage;
- retention and usage of dashboards/debugging/integrations.

Exact target values are deferred until market validation and MVP definition.

## 13. Product principles

1. **Facts before dashboards.** Dashboards are derived views, not sources of truth.
2. **Traceability over magic.** Every important number should have a path back to source facts.
3. **Simple UX over configuration density.** Complexity belongs in the system, not in the operator's workflow.
4. **Multi-tenancy from the model.** Tenant boundaries are not an afterthought.
5. **Observable by default.** Tracking failures must be diagnosable.
6. **Provider-neutral core.** Integrations adapt to Trackfy, not vice versa.
7. **Deterministic financial semantics.** Revenue, cost and profit calculations must be explicit.
8. **Recomputation over hidden mutation.** Derived analytics should be reproducible.
9. **Contracts before implementation.** Event and integration behavior must be defined before production code.

## 14. Deferred decisions

The following remain open and require dedicated specifications:

- canonical event envelope;
- event taxonomy;
- identity-resolution algorithm;
- browser/server conflict rules;
- deduplication and idempotency semantics;
- timestamp authority and late-arrival handling;
- attribution windows and model formulas;
- currency and financial precision rules;
- exact RBAC permission matrix;
- exact Organization/Workspace/Site hierarchy semantics;
- analytics storage strategy;
- integration protocol details;
- MVP scope and release criteria.

## 15. Next specifications

The product definition in this document feeds the following artifacts, in order:

1. Domain Model v0.1
2. Event and Tracking Contracts
3. Attribution Model
4. Architecture
5. Multi-tenancy + RBAC specification
6. MVP Specification
7. Engineering Harness

No production implementation should be considered complete until the relevant contracts and acceptance tests exist.
