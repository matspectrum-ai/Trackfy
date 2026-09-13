# Trackfy — Domain Model v0.1

**Status:** Draft
**Version:** 0.1

This document defines the conceptual domain vocabulary required by the Product Spec. It is not a database schema. Persistence choices, keys, indexes and physical normalization belong to later engineering design.

## 1. Domain map

```text
User
  │
  ├── Organization Membership ──→ Organization
  │                                │
  │                                ├── Workspace
  │                                │     │
  │                                │     ├── Site
  │                                │     │     │
  │                                │     │     ├── Visitor
  │                                │     │     │     └── Session
  │                                │     │     │          └── Click / Event
  │                                │     │     │
  │                                │     │     └── Tracking Configuration
  │                                │     │
  │                                │     ├── Campaigns / Sources
  │                                │     ├── Ad Accounts
  │                                │     ├── Sales Integrations
  │                                │     ├── Webhooks / API Keys
  │                                │     └── Dashboards / Reports / Rules
  │                                │
  │                                └── Members / Roles
  │
  └── Personal Workspace (optional product behavior; final semantics deferred)
```

The behavioral journey is:

```text
Acquisition
   ↓
Click
   ↓
Session
   ↓
Event
   ↓
Lead
   ↓
Checkout
   ↓
Order
   ↓
Payment / Order Lifecycle
   ↓
Attribution
   ↓
Revenue / Cost / Profit Analytics
```

## 2. Identity and tenancy

### User
A human account capable of authenticating and participating in one or more organizations.

A User is not the owner of tenant data by default. Tenant ownership and access are established through membership.

### Organization
The top-level tenant boundary for collaborative business data.

An Organization owns or scopes resources such as Workspaces, Members, Sites, integrations, dashboards and API credentials.

### Organization Membership
Association between a User and an Organization, carrying a role and the effective access scope.

### Workspace
A working context inside an Organization. A Workspace is intended to isolate operational data, tracking configurations and reporting contexts without requiring a separate Organization.

The exact relationship among Organization, Workspace and Project remains a product decision, but the first-class Workspace boundary is required.

### Role
A named permission profile applied through membership. Initial candidates are:

- Owner
- Admin
- Analyst
- Media Buyer
- Viewer
- Developer

The exact permission matrix is deferred to the Multi-tenancy + RBAC specification.

## 3. Acquisition domain

### Site
A tracked web property or application surface associated with a Workspace.

A Site provides the context for browser-side tracking and visitor/session collection.

### Tracking Source
A normalized representation of where traffic originated. It may derive from UTM parameters, ad-platform click identifiers, referral context or other configured acquisition metadata.

### Campaign
A marketing campaign or campaign identity used for performance analysis. Campaign dimensions may originate from ad platforms, UTMs or manually configured mappings.

### Ad Account
An external advertising account connected to Trackfy for spend and campaign data.

### Tracking Configuration
The configuration that tells Trackfy how a Site should emit, accept and associate tracking events.

### Tracking Domain
A domain or endpoint used as part of the first-party/server-side tracking path. Exact infrastructure semantics are deferred.

## 4. Journey and identity domain

### Visitor
A Trackfy identity representing a browser/device-level actor or other stable tracking identity within the permitted measurement model.

Visitor identity must not be assumed to equal a person identity.

### Session
A bounded interaction period associated with a Visitor and Site. A Session groups events that occur within a defined sessionization policy.

The session timeout and exact re-identification semantics are deferred.

### Click
An acquisition interaction containing source/campaign information and, where available, click identifiers such as `fbclid`, `gclid`, `ttclid` or a Trackfy `click_id`.

A Click can be linked to a Session and downstream events.

### Event
An immutable observation of something that occurred in the tracking or business lifecycle.

Examples include:

- page view;
- click;
- lead;
- checkout started;
- purchase;
- postback received;
- order status changed;
- refund recorded.

An Event has an origin, occurrence time, ingestion time, identity/context and a typed payload. The canonical envelope is defined later in Event/Tracking Contracts.

### Lead
A business milestone indicating that an identifiable acquisition journey reached a lead/conversion-intent state.

A Lead is not necessarily an Order.

### Checkout
A business milestone representing progression into checkout. It may occur before an Order exists.

## 5. Commerce and financial domain

### Order
A business transaction representing a purchase/conversion attempt or confirmed transaction received from a sales source.

An Order has a lifecycle and may have monetary components and external identifiers.

### Order Status
Initial conceptual states:

`pending`, `approved`, `rejected`, `cancelled`, `refunded`, `chargeback`, `partially_refunded`.

Provider-specific states must be normalized to this canonical lifecycle while preserving source provenance.

### Payment Lifecycle
The financial state history associated with an Order. It represents status transitions rather than treating the current status as the only fact of interest.

Detailed state-machine rules, allowed transitions and reversal behavior are deferred.

### Refund
A negative financial adjustment against an Order. Refunds may be partial or full and must remain distinguishable from cancellations that do not necessarily represent the same financial outcome.

### Fee
A cost charged against a transaction or integration outcome, such as payment processing or platform fees.

### Commission
A revenue share or commission amount associated with an Order or commercial arrangement.

### Product Cost
A direct cost associated with fulfilling the sale. Product representation itself is not a core Trackfy product catalog domain in v0.1; this concept exists to support profitability calculations.

### Ad Spend
Marketing expenditure associated with an advertising source, campaign, ad set or account over a defined period.

### Revenue
Money associated with Orders, separated into distinct analytical concepts such as gross, approved, pending and adjusted revenue.

### Profit
A derived financial metric. It is not itself a raw source fact.

Initial conceptual formula:

`Gross Revenue - Refunds - Fees - Commissions - Product Cost = Net Revenue`

`Net Revenue - Ad Spend = Net Profit`

The source material defines this separation explicitly; precision, timing and currency rules are deferred. fileciteturn3file1L13-L24

## 6. Attribution domain

### Attribution Touchpoint
An interaction eligible to receive attribution credit for a conversion. A touchpoint may be a Click or another qualifying acquisition event depending on the future model.

### Attribution Window
The temporal range within which touchpoints may influence a conversion. The exact window is deferred.

### Attribution Model
A deterministic rule set that distributes conversion or revenue credit across eligible touchpoints.

Initial candidate models:

- First Touch
- Last Touch
- Linear / Multi-touch
- Future custom models

### Attribution Result
A derived record explaining how a conversion/order/revenue outcome was assigned to one or more touchpoints under a named model and set of rules.

An Attribution Result must be reproducible from canonical source facts and model configuration.

## 7. Integration domain

### Sales Integration
Connection to an external commerce, checkout or sales provider that can produce conversion and lifecycle information.

### Advertising Integration
Connection to an advertising provider used to retrieve or synchronize campaign/account/spend information.

### Webhook Endpoint
An authenticated inbound or outbound integration surface for asynchronous events.

### API Key
A credential representing a programmatic integration identity and scope. Secrets must not be exposed to unauthorized users or clients.

### External Identifier
An identifier originating in another platform, such as an external order ID, campaign ID or customer ID. External identifiers must retain their source/provider context.

### Integration Event
An external event received from a provider before or during normalization into canonical Trackfy facts.

## 8. Analytics domain

### Metric
A deterministic derived value computed from canonical facts and dimensions.

### Dashboard
A persisted analytical view/configuration over Trackfy metrics and dimensions.

### Funnel
An ordered analytical view of milestone events across the journey, such as visit → lead → checkout → purchase.

### Cohort
A group of visitors/customers/orders sharing a defined characteristic or time-based property.

### Report
A reusable analytical representation with filters, dimensions and metrics.

### Rule
A future automation definition evaluated against events or metrics to produce an action, alert or recommendation.

### Alert
A notification generated when a configured rule or system-health condition is satisfied.

## 9. Observability domain

### Event Trace
The correlated sequence of events representing a journey or conversion path.

### Event Debugger
The product interface used to inspect an Event Trace and processing state.

### Processing State
The status of an event as it moves through ingestion and downstream processing. Initial conceptual values may include accepted, rejected, duplicated, unmatched, processed and failed; exact state vocabulary is deferred to event contracts.

### Match
A successful association between independent records using available identity signals, such as an Order matched to a Visitor/Session/click path.

### Mismatch / Unmatched Event
A valid event that cannot be associated with the expected journey or business entity under the current identity rules.

## 10. Source-of-truth hierarchy

The domain requires a distinction between canonical facts and derived representations.

### Canonical facts
Events, external source records, lifecycle transitions and other durable observations that should remain available as evidence.

### Derived state
Current lifecycle summaries, sessions, attribution results, aggregates and dashboards calculated from canonical facts and deterministic rules.

### Presentation
UI representations of derived state. Presentation must never become an independent source of truth.

This separation follows the established principle that events are immutable facts and analytics should be recomputable. fileciteturn5file0L1-L2

## 11. Key relationships

```text
User
  └──< Organization Membership >── Organization
                                      └──< Workspace
                                           ├──< Site
                                           │    └──< Visitor
                                           │         └──< Session
                                           │              ├──< Click
                                           │              └──< Event
                                           │
                                           ├──< Campaign
                                           ├──< Ad Account
                                           ├──< Sales Integration
                                           ├──< Advertising Integration
                                           ├──< Webhook Endpoint
                                           ├──< API Key
                                           ├──< Dashboard
                                           └──< Rule

Visitor / Session / Click / Event
              │
              └──────────────→ Lead / Checkout / Order
                                      │
                                      ├──< Payment Lifecycle Transition
                                      ├──< Refund
                                      ├──< Fee / Commission / Cost
                                      └────────────→ Attribution Result

Ad Account / Campaign
              │
              └────────────→ Ad Spend

Canonical Facts
      │
      ├──→ Attribution Results
      ├──→ Financial Metrics
      ├──→ Funnels / Cohorts
      └──→ Dashboards / Reports
```

## 12. Invariants to preserve

1. A tenant-owned object must have an unambiguous authorization scope.
2. Raw event evidence must not be silently rewritten when derived state changes.
3. An Order is distinct from a Click, Lead and Checkout.
4. Order lifecycle transitions must be observable and source-aware.
5. Revenue must not be treated as equivalent to profit.
6. Pending, approved and refunded outcomes must remain distinguishable.
7. Attribution must identify the model and inputs used to produce its result.
8. External identifiers must retain provider/source context.
9. Duplicate delivery must not silently create duplicate business outcomes.
10. Analytics must be derivable from canonical facts rather than from UI state.

## 13. Deferred domain decisions

The following are intentionally unresolved:

- whether Project remains a first-class domain object or is removed in favor of Workspace/Site;
- exact Workspace isolation semantics;
- whether a Site belongs directly to Workspace or can be shared;
- exact Visitor identity mechanism and retention;
- sessionization policy;
- canonical event taxonomy;
- relationship cardinalities for Order, Customer and Payment;
- payment provider versus sales-provider ownership of lifecycle truth;
- currency model and multi-currency reporting;
- exact cost model;
- attribution touchpoint eligibility;
- attribution windows;
- exact RBAC permissions;
- API key scopes;
- retention and deletion policy.

## 14. Next step

The next artifact should be **Event and Tracking Contracts v0.1**. It must turn the conceptual Event/Visitor/Session/Click/Order boundaries above into explicit machine-readable contracts, including identifiers, deduplication, idempotency, timestamps, browser/server ingestion and source-of-truth rules.
