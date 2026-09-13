# Trackfy Contract Registry

This directory contains the machine-oriented boundaries that implementation and tests must follow.

## Contract authority

| Contract | Path | Authority |
|---|---|---|
| Canonical Event v1 | `event/v1/event.schema.json` | Event envelope shape and field constraints |
| Ingestion API v1 | `http/v1/ingestion.openapi.yaml` | HTTP transport and response semantics |
| Browser SDK v1 | `browser-sdk/v1/browser-sdk-contract.md` | Browser producer behavior |
| Debugger Trace v1 | `debugger/v1/event-trace-contract.md` | Debugger read-model semantics |

Higher-level product semantics remain governed by the documents under `docs/product`, `docs/attribution`, `docs/architecture`, `docs/security` and `docs/mvp`.

## Rules

1. Code must not silently diverge from a published contract.
2. A breaking contract change requires a new version.
3. Every contract must have positive and negative test coverage.
4. Fixtures under `tests/fixtures` are executable examples of expected semantics.
5. Provider-specific schemas belong to integration adapters, not the canonical event schema.
6. A schema validates structure; business invariants additionally require domain tests.
7. Contract tests must run in CI before merge.

## Required verification layers

```text
Schema validation
      ↓
Contract tests
      ↓
Domain/invariant tests
      ↓
Integration tests
      ↓
End-to-end tests
```

## First required scenarios

- valid browser funnel;
- browser/server duplicate conversion;
- cross-tenant rejection;
- idempotency replay;
- malformed event rejection;
- missing optional acquisition data;
- late-arriving event;
- downstream delivery failure without canonical event failure.

## Implementation rule

The first implementation slice must consume these contracts rather than inventing a parallel internal protocol.
