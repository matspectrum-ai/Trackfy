# Trackfy Event Trace Contract v1

**Status:** Draft  
**Version:** 1  
**Purpose:** Define the read model consumed by the Event Debugger.

## 1. Principle

The debugger is a projection over canonical events, correlation edges, processing outcomes and downstream delivery records. It is never a source of truth.

## 2. Trace shape

Conceptual response:

```json
{
  "trace_id": "trace_01J...",
  "tenant": {
    "organization_id": "org_...",
    "workspace_id": "ws_...",
    "site_id": "site_..."
  },
  "subject": {
    "visitor_id": "vis_...",
    "session_id": "ses_...",
    "order_id": null
  },
  "events": [],
  "correlations": [],
  "deliveries": [],
  "health": {}
}
```

## 3. Event projection

Each event item should expose:

- `event_id`;
- event name/version;
- occurred/received timestamps;
- source/provider;
- tenant scope;
- visitor/session/click IDs where available;
- acquisition summary;
- payload summary;
- processing state;
- validation/correlation status;
- provenance reference.

Sensitive raw payloads are not returned by default. Access to raw evidence is separately authorized.

## 4. Correlation projection

A correlation edge describes how two records were related.

Conceptual fields:

```json
{
  "from": "evt_...",
  "to": "ord_...",
  "method": "click_id",
  "confidence": "deterministic"
}
```

Allowed evidence classes for v1:

- explicit Trackfy identity;
- provider/external identifier + integration scope;
- Trackfy click ID;
- session ID;
- visitor ID;
- configured deterministic signal.

Heuristic similarity is not a canonical v1 correlation method.

## 5. Delivery projection

Each downstream delivery item should expose:

- destination;
- integration/account reference;
- canonical `event_id`;
- delivery state;
- attempts;
- first/last attempt times;
- provider event/request ID when available;
- safe error category;
- next retry time when applicable.

Provider secrets and sensitive request bodies are never returned.

## 6. Health projection

The debugger may show derived indicators:

```text
received
validated
correlated
processed
attributed
routed
delivered
```

These indicators must link to underlying event/delivery state and must not masquerade as raw facts.

## 7. Authorization

A debugger query is valid only when the requesting principal has access to the trace tenant scope.

A user who can view Workspace A must not infer or retrieve traces belonging to Workspace B.

## 8. Query expectations

The read model should support:

- trace by event ID;
- trace by visitor ID;
- trace by session ID;
- trace by order ID;
- trace around a time range;
- filter by processing state;
- filter by source/provider;
- filter by integration delivery state.

Queries should not require unrestricted scans of raw event history for normal debugger operation.

## 9. Acceptance criteria

### DBG-AC-01 — Tenant isolation
An authorized user receives only traces within the accessible tenant/workspace scope.

### DBG-AC-02 — Event identity
The debugger displays the canonical `event_id` exactly as stored.

### DBG-AC-03 — Timeline integrity
Events are displayed in canonical occurrence order with arrival/processing metadata available for diagnosis.

### DBG-AC-04 — Duplicate visibility
Duplicate deliveries are visible as processing evidence without appearing as additional canonical facts.

### DBG-AC-05 — Correlation explanation
The debugger identifies the deterministic correlation method used to associate records.

### DBG-AC-06 — Delivery isolation
A downstream provider failure is visible as a delivery failure and does not mark the canonical event as failed.

### DBG-AC-07 — Sensitive data
Default debugger responses do not expose provider secrets or unnecessary sensitive payload content.
