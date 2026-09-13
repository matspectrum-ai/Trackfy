# Trackfy Browser SDK Contract v1

**Status:** Draft  
**Version:** 1  
**Purpose:** Define the stable browser-side producer contract without coupling product behavior to a particular JS implementation.

## 1. Responsibilities

The SDK is responsible for collection and transport preparation, not authorization or business truth.

It may:

- initialize against a site-scoped public identifier;
- establish or reuse permitted first-party visitor identity;
- establish or reuse session context;
- read acquisition parameters from the current navigation context;
- preserve provider click identifiers;
- generate `event_id` values;
- emit standard or custom events;
- attach browser-safe context;
- queue transiently when documented;
- expose local diagnostics.

It must never contain server credentials or provider secrets.

## 2. Public initialization

Conceptual API:

```ts
trackfy.init({
  siteKey: "site_public_...",
  endpoint: "https://tracking.example.com/v1/ingest",
  autoPageView: true
})
```

The exact runtime API is an implementation decision. The semantic requirements below are not.

## 3. Event emission

Conceptual API:

```ts
trackfy.track("page_view", {
  path: window.location.pathname
})
```

The SDK must generate or preserve a unique `event_id` for each logical event assertion.

For browser/server copies of the same conversion, the application must be able to explicitly supply the logical event identity or correlation identifier required by the server contract.

## 4. Automatic context

The SDK may automatically provide:

- current URL/path/title;
- locale/timezone;
- permitted device/browser context;
- visitor/session IDs;
- UTM fields;
- provider click IDs;
- referrer where available;
- consent state provided by the application.

Automatic collection must remain bounded by the site's configuration and applicable privacy policy.

## 5. Data Layer / GTM integration

Trackfy must provide a deterministic mapping from GTM data-layer events to the canonical event envelope.

Example:

```js
dataLayer.push({
  event: "purchase",
  order_id: "ord_123",
  value: 199.90,
  currency: "BRL"
})
```

The adapter must preserve the source event name and map explicit values without silently changing business meaning.

## 6. Transport behavior

The SDK must:

1. use HTTPS for production transport;
2. send to a configured first-party/Trackfy endpoint;
3. avoid blocking critical page interaction where possible;
4. tolerate duplicate delivery;
5. distinguish retryable transport failures from contract rejection;
6. avoid retrying deterministic 4xx validation failures indefinitely;
7. avoid exposing raw secrets in error output.

## 7. Browser/server coexistence

The browser SDK is not authoritative for business outcomes such as approved payment.

Example:

```text
Browser purchase assertion
          +
Server provider assertion
          ↓
       Trackfy
          ↓
 canonical correlation + deduplication
```

A browser `purchase` event must not by itself force an Order into `approved` status.

## 8. First-party identity

The SDK may persist Trackfy visitor/session context using an approved first-party mechanism.

It must:

- allow identity reset/invalidation according to site policy;
- never claim visitor identity is human identity;
- preserve tenant/site scope;
- avoid silently merging identities across unrelated domains.

## 9. Consent boundary

The SDK must expose a consent-aware initialization/collection path.

Consent state is contextual input, not a legal authorization guarantee.

The SDK must not invent consent.

## 10. Debug mode

A local/debug mode should expose:

```text
queued event
↓
event_id
↓
payload summary
↓
transport result
↓
server request ID
```

Debug mode must redact configured sensitive fields.

## 11. Failure behavior

The SDK must never convert a tracking failure into an application failure by default.

A failed `page_view` must not crash the host application.

A failed `purchase` tracking request must not cancel or alter the host application's Order transaction.

## 12. Acceptance criteria

### SDK-AC-01 — Site scope
A site-scoped browser producer cannot ingest into an unrelated site/tenant.

### SDK-AC-02 — Event identity
Each logical event receives one stable `event_id` before successful transmission.

### SDK-AC-03 — Acquisition preservation
UTM and provider click identifiers are preserved as supplied by the browser until server-side normalization.

### SDK-AC-04 — Secret isolation
No server/provider secret is present in the browser bundle or runtime configuration.

### SDK-AC-05 — Non-blocking failure
A rejected or unavailable tracking endpoint does not crash or block normal host application behavior.

### SDK-AC-06 — Browser/server compatibility
A browser event and a server event representing the same logical conversion can be correlated without creating a second canonical event when the defined deduplication key is shared.

### SDK-AC-07 — Debug visibility
A developer can identify the generated event ID and transport result in development mode.
