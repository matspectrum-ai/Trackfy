# Trackfy — Attribution Model v0.1

**Status:** Draft  
**Version:** 0.1  
**Scope:** Attribution of conversions, Orders and revenue to eligible acquisition touchpoints using canonical tracking facts and deterministic models.

This document defines the semantic attribution contract. It does not define database tables, SQL, APIs or a specific analytics engine.

## 1. Purpose

Trackfy attribution answers:

> Given a conversion or Order, which acquisition touchpoints should receive credit, under which model, for which economic amount, and based on which evidence?

Attribution is a derived analytical result. It must never rewrite canonical tracking events, Orders or payment lifecycle facts.

The same canonical facts and model configuration must reproduce the same result.

## 2. Separation of concerns

```text
Tracking
  ↓
records what happened
  ↓
Canonical facts
  ↓
Attribution eligibility
  ↓
Attribution model
  ↓
Credit allocation
  ↓
Revenue / conversion analytics
```

Tracking does not decide credit. Attribution does not invent missing tracking facts. Financial analytics does not modify attribution evidence.

## 3. Core concepts

### Conversion
A business outcome eligible for attribution. In v0.1 the primary conversion target is an Order or another explicitly configured conversion event.

### Touchpoint
An interaction eligible to receive attribution credit. A Touchpoint may originate from a Click or another explicitly qualifying acquisition event.

### Attribution window
The maximum temporal interval in which a touchpoint may influence a conversion.

### Attribution model
A deterministic rule defining how eligible touchpoints receive conversion or economic credit.

### Attribution result
The immutable analytical output of applying one model/configuration to canonical facts at a defined point in time.

### Attribution scope
The Organization/Workspace/Site context in which the model and eligible facts are interpreted.

## 4. Attribution invariant

For an attribution run:

```text
eligible touchpoints
        +
conversion facts
        +
model configuration
        +
window rules
        +
eligibility rules
        ↓
exact reproducible allocation
```

The result must retain enough metadata to explain:

- conversion/order identity;
- selected model;
- model version;
- attribution window;
- eligible touchpoints;
- excluded touchpoints and exclusion reasons where relevant;
- allocated credit;
- economic basis of the allocation;
- generation timestamp/version.

## 5. Touchpoint eligibility

A touchpoint is eligible only when it belongs to the same authorized attribution scope and satisfies the configured temporal and identity rules.

Initial eligibility requirements:

1. Touchpoint is a valid canonical fact.
2. Touchpoint has an acquisition origin or explicitly qualifying event type.
3. Touchpoint belongs to the same Site/Workspace attribution scope unless cross-site attribution is explicitly configured.
4. Touchpoint occurred before or at the conversion according to model rules.
5. Touchpoint falls inside the attribution window.
6. Touchpoint has sufficient identity/correlation evidence.
7. Touchpoint is not explicitly excluded by configuration or privacy/consent policy.

A missing touchpoint must remain missing. Attribution must not fabricate a source simply to avoid an unattributed conversion.

## 6. Identity continuity

Attribution may use:

```text
click_id
session_id
visitor_id
customer_id
external_customer_id
external_order_id
```

The strongest deterministic relationship available should be preferred.

Conceptually:

```text
Order
  ↓
Customer identity
  ↓
Visitor / Session
  ↓
Click / Touchpoint
```

Weak or heuristic identity matches must not silently receive the same semantic status as explicit deterministic relationships.

## 7. Attribution windows

The window is part of the model configuration and therefore part of the attribution result.

The engine must support configurable windows rather than hard-coding one global value.

Candidate configurations include:

```text
same-session
N hours
N days
N rolling days before conversion
custom organization policy
```

A touchpoint outside the configured window is ineligible even if an identity relationship exists.

The default production window is intentionally deferred until sufficient product requirements and customer use cases are established.

## 8. Baseline models

### 8.1 First Touch

100% of conversion credit goes to the earliest eligible touchpoint in the attribution window.

```text
A → B → C → Conversion
A = 100%
B = 0%
C = 0%
```

### 8.2 Last Touch

100% of conversion credit goes to the latest eligible touchpoint before conversion.

```text
A → B → C → Conversion
A = 0%
B = 0%
C = 100%
```

Direct/organic traffic must not automatically erase a paid touchpoint unless the configured eligibility rules explicitly make it the winning touchpoint.

### 8.3 Linear Multi-touch

Credit is distributed equally across all eligible touchpoints.

For `n` eligible touchpoints:

`credit_per_touchpoint = 1 / n`

Example:

```text
A → B → C → Conversion
A = 33.33%
B = 33.33%
C = 33.33%
```

Rounding must never cause allocated economic credit to exceed or fall short of the modeled total. The exact precision/rounding policy is part of the financial analytics contract.

## 9. Model extensibility

The attribution engine must be designed to support additional deterministic models without changing canonical event semantics.

Potential future models:

- position-based;
- time-decay;
- weighted custom multi-touch;
- data-driven models when explicitly supported;
- organization-defined rules.

A future model must define eligibility, weighting, normalization, tie-breaking and rounding explicitly.

## 10. Conversion target

The attribution engine must distinguish event types from economic conversion targets.

Example:

```text
checkout_started → conversion event candidate
order_created    → commerce fact
approved order   → economic conversion candidate
refund           → economic adjustment
```

The product may expose several analytical views, but the underlying facts remain distinct.

## 11. Pending versus approved revenue

Trackfy must never equate conversion occurrence with approved economic revenue.

An Order may be:

```text
pending
approved
rejected
cancelled
refunded
partially_refunded
chargeback
```

Attribution can establish which touchpoints contributed to an Order even while its economic state is pending.

Financial reporting must distinguish at least:

```text
Attributed Gross Revenue
Attributed Approved Revenue
Attributed Pending Revenue
Attributed Refunded Amount
Attributed Net Revenue
```

The attribution result itself should identify the underlying conversion/order; financial projections can recalculate economic values from lifecycle facts.

## 12. Revenue allocation

There are two distinct concepts:

### Conversion credit
A normalized share such as `0.25` or `25%`.

### Economic credit
The monetary amount represented by that share under a selected revenue basis.

Example:

```text
Order gross = R$400
Model = Linear
Eligible touchpoints = 4

Each touchpoint:
25% conversion credit
R$100 gross attributed credit
```

The result must state which economic basis was used, such as gross or approved revenue.

## 13. Refunds and reversals

Refunds must not delete the original attribution result.

Instead:

```text
Original Order
   ↓
Original attribution
   ↓
Refund fact
   ↓
Derived negative economic adjustment
```

A full refund may reduce net attributed revenue to zero while preserving the evidence that the conversion occurred and which touchpoints originally received credit.

A partial refund reduces the economic amount proportionally according to the financial rules.

Chargebacks and cancellations must remain semantically distinct from refunds even when they reduce economic value.

## 14. Fees and commissions

Attribution credit and profitability are different calculations.

Example:

```text
Order gross                         500
Attributed gross credit             250
Fees                                 20
Commission                           30
Product cost                         80
Ad spend attribution                100
---------------------------------------
Derived profit basis                 20
```

The exact profitability formula belongs to the financial analytics specification.

Attribution should provide the links needed for financial allocation without embedding accounting rules into the touchpoint model.

## 15. Ad spend attribution

Ad spend is not automatically equivalent to conversion revenue and must be sourced independently from advertising/account data.

Trackfy must allow analytics to connect:

```text
Ad account
  ↓
Campaign / ad dimensions
  ↓
Spend
  ↓
Attributed conversions / revenue
  ↓
ROAS / profit analytics
```

Where spend cannot be deterministically joined to an attributed touchpoint, it must remain separately reported rather than silently fabricated.

## 16. Multiple identifiers and source precedence

A conversion may contain several valid acquisition signals:

```text
utm_campaign
fbclid
gclid
click_id
session_id
visitor_id
```

Attribution must preserve all evidence and apply explicit precedence rules rather than replacing one signal with another during ingestion.

The canonical `click_id` is the preferred Trackfy touchpoint identity when available, while provider click IDs remain evidence for correlation and downstream integrations.

## 17. Direct / organic traffic

Direct or organic classification is a derived interpretation, not proof that no paid acquisition occurred.

A later direct visit must not erase an earlier eligible paid touchpoint.

The selected attribution model determines whether direct/organic touchpoints can receive credit.

## 18. Multiple conversions

Each conversion/order is attributed independently unless a higher-level aggregation explicitly states otherwise.

A visitor may produce multiple Orders, and each Order may have a different eligible touchpoint set according to its own conversion timestamp and attribution window.

## 19. Reprocessing and reproducibility

Attribution results must be reproducible.

When canonical facts or model configuration change, Trackfy may produce a new attribution result version rather than mutating historical output without traceability.

Conceptually:

```text
Facts v1 + Model v1 → Attribution Result v1
Facts v1 + Model v2 → Attribution Result v2
```

Historical results must retain:

- model name;
- model version;
- configuration snapshot/hash where appropriate;
- attribution window;
- input conversion identity;
- input touchpoint identities;
- generated-at timestamp;
- result version.

## 20. Conflict and missing data

Attribution must surface ambiguity rather than conceal it.

Examples:

```text
Order with no touchpoint
→ unattributed

Multiple conflicting external identifiers
→ correlation conflict

Touchpoint outside window
→ excluded

Visitor exists but no eligible click
→ unattributed / organic-direct classification if supported
```

The engine must not manufacture a winning touchpoint solely to achieve 100% attribution coverage.

## 21. Attribution confidence

A future attribution result may expose evidence quality, but confidence must never disguise uncertainty.

Proposed conceptual categories:

```text
explicit
strong-deterministic
deterministic-recovered
unmatched
conflicted
```

A probabilistic or heuristic score should only be introduced as a separate, explicitly labeled capability.

## 22. Explainability

Every attribution result shown to a user should be explainable through a path such as:

```text
Order #123
   ↓
Session #ABC
   ↓
Click #XYZ
   ↓
Meta / Campaign A
   ↓
Last Touch
   ↓
100% credit
   ↓
R$ 500 attributed gross
```

For multi-touch:

```text
Click A  → 40%
Click B  → 35%
Click C  → 25%
```

The interface should be able to answer why a touchpoint was included or excluded.

## 23. Re-attribution

When new authoritative information arrives, such as a late server-side Order event, Trackfy may recompute attribution.

The recomputation must:

1. preserve canonical facts;
2. record the reason for recomputation;
3. produce a traceable new result/version;
4. allow downstream analytics to know which result is current under the active policy.

## 24. Attribution and downstream integrations

The attribution engine produces canonical analytical outcomes. Downstream adapters may use those outcomes to send conversion information to advertising or analytics platforms.

```text
Canonical facts
      ↓
Attribution
      ↓
Selected conversion representation
      ↓
Meta / Google / TikTok / GA4 / CRM
```

A downstream platform's attribution model must not silently replace Trackfy's internal attribution result.

## 25. Acceptance criteria

### AC-01 — First touch
Given three eligible touchpoints before an Order, First Touch assigns 100% credit to the earliest touchpoint.

### AC-02 — Last touch
Given three eligible touchpoints before an Order, Last Touch assigns 100% credit to the latest eligible touchpoint.

### AC-03 — Linear
Given four eligible touchpoints, Linear assigns equal credit and economic allocation sums exactly to the modeled conversion amount subject to defined monetary rounding.

### AC-04 — Window
Given a touchpoint outside the configured attribution window, that touchpoint receives zero credit and is marked excluded.

### AC-05 — No fabrication
Given an Order with no eligible touchpoints, the result is unattributed rather than assigned to an arbitrary source.

### AC-06 — Pending order
Given a pending Order with valid touchpoints, attribution may be computed while approved revenue remains zero under approved-revenue reporting.

### AC-07 — Refund
Given an attributed Order followed by a full refund, the original attribution remains traceable while derived net economic credit is reduced according to the financial rules.

### AC-08 — Partial refund
Given a partial refund, attributed net economic value decreases by the applicable refunded amount without deleting the original conversion.

### AC-09 — Determinism
Given identical canonical facts, model version and configuration, repeated attribution runs produce the same logical allocation.

### AC-10 — Reprocessing
Given a model configuration change, Trackfy produces a traceable new attribution result rather than silently overwriting the prior result.

### AC-11 — Explainability
Given an attribution result, an authorized user can inspect the conversion, eligible touchpoints, model, window and resulting credit allocation.

### AC-12 — Provider independence
Given provider-specific identifiers from different advertising platforms, internal attribution semantics remain provider-neutral while provenance is preserved.

## 26. Deferred decisions

The following remain intentionally open:

- production default attribution window;
- exact direct/organic eligibility policy;
- whether `engagement`, `lead` and `checkout_started` can independently become conversion targets;
- exact position-based/time-decay formulas;
- custom weighting UI;
- cross-site/cross-workspace attribution policy;
- customer identity stitching policy;
- probabilistic attribution, if ever introduced;
- exact monetary rounding/precision rules;
- ad-spend allocation methodology;
- treatment of commissions and product costs in attribution-facing metrics;
- retention policy for historical attribution versions.

## 27. Next step

The next specification should define **Architecture v0.1**, separating Control Plane, Tracking/Data Plane, Analytics, Integrations and Dashboard while preserving the canonical-event and deterministic-attribution boundaries established here.
