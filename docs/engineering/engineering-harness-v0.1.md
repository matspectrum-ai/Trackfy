# Trackfy — Engineering Harness v0.1

**Status:** Draft
**Version:** 0.1

## 1. Purpose

The Engineering Harness is the operating system for developing Trackfy safely with human engineers and coding agents. It converts product intent into observable, testable and verifiable engineering work.

The harness is not a collection of prompts. It is a set of repository rules, artifacts, checks and feedback loops.

## 2. Source-of-truth chain

```text
Product Spec
    ↓
Domain Model
    ↓
Contracts
    ↓
Architecture / Security
    ↓
MVP Spec
    ↓
Implementation Spec
    ↓
Tests / Evals
    ↓
Code
```

Code must not become the implicit source of truth for product behavior.

## 3. Work unit

Every non-trivial engineering change should have a bounded work unit containing:

- problem statement;
- governing specification;
- scope and non-scope;
- acceptance criteria;
- affected contracts;
- security/tenant impact;
- test strategy;
- verification evidence.

A work unit can be represented by an issue, spec or task artifact depending on workflow maturity.

## 4. Development loop

```text
OBSERVE
  ↓
read repo + specs + current behavior
  ↓
SPECIFY
  ↓
define observable behavior and acceptance criteria
  ↓
PLAN
  ↓
choose minimal coherent change
  ↓
ACT
  ↓
implement
  ↓
VERIFY
  ↓
tests + static checks + runtime checks
  ↓
ITERATE
```

Agents must not jump directly from an ambiguous request to broad implementation.

## 5. Test pyramid

### Unit tests

Use for deterministic domain behavior, parsers, normalization, attribution calculations and other logic with clear inputs/outputs.

### Contract tests

Use for event schemas, API boundaries, integration adapters and versioned interfaces.

### Integration tests

Use for database, queue, storage and external-system boundaries where unit tests cannot establish correctness.

### End-to-end tests

Use only for critical user journeys and cross-boundary behavior. Do not turn every unit behavior into a slow end-to-end test.

### Invariant/property tests

Use when broad input spaces or state transitions make example-only tests insufficient, especially for event idempotency, deduplication, lifecycle transitions and attribution conservation.

## 6. Contract-driven development

Before implementing a public boundary, define:

- request shape;
- response/acknowledgement semantics;
- validation behavior;
- error model;
- versioning policy;
- authentication/authorization boundary;
- idempotency behavior when applicable;
- observable telemetry.

Implementation follows the contract rather than defining it accidentally.

## 7. Acceptance-criteria traceability

Each acceptance criterion should map to one or more verification mechanisms.

Example:

```text
AC: duplicate purchase must not create a second business outcome
  ↓
contract test
  + integration test
  + idempotency invariant
```

A ticket is not complete merely because code exists; the acceptance criteria must have evidence.

## 8. Evaluation layer

Trackfy uses three complementary evaluation types.

### Deterministic evals

Machine-checkable expectations for schemas, IDs, state transitions, metrics and API semantics.

### Scenario evals

Representative journeys such as:

```text
ad click → landing → lead → checkout → order pending → approved
```

or:

```text
browser purchase + server purchase → one canonical conversion
```

### Regression corpus

A versioned set of previously failing or high-risk scenarios that must remain green as the system evolves.

## 9. Tracking-specific verification

Minimum critical invariants include:

1. one canonical `event_id` cannot create duplicate canonical facts;
2. `occurred_at` and `received_at` remain distinct;
3. tenant scope cannot cross Organization/Workspace boundaries;
4. browser and server copies of one conversion can be deduplicated when contractually linked;
5. provider-specific identifiers retain source scope;
6. late events remain processable;
7. canonical evidence can be replayed into derived state;
8. attribution cannot assign credit without eligible evidence;
9. downstream delivery failure cannot invalidate canonical tracking;
10. financial projections cannot silently replace source lifecycle facts.

## 10. Multi-tenant verification

Every feature touching customer data must include negative authorization tests.

At minimum test:

```text
Organization A → access A = allowed
Organization A → access B = denied
Workspace A → access sibling workspace = denied unless explicitly permitted
Role without permission → operation = denied
Deleted/revoked credential → protected operation = denied
```

Workers and background jobs must carry explicit tenant context rather than relying on ambient process state.

## 11. Agent workflow

A coding agent should normally execute this sequence:

```text
1. Inspect repository and governing docs.
2. Identify exact work unit and acceptance criteria.
3. Find affected contracts and invariants.
4. Produce a concise implementation plan.
5. Implement the smallest coherent change.
6. Run focused tests.
7. Run broader regression checks.
8. Inspect failures rather than suppress them.
9. Update documentation/contracts when behavior changed.
10. Summarize verification evidence.
```

The agent should stop and surface uncertainty when the requested behavior conflicts with a specification or security invariant.

## 12. Change-size discipline

Prefer changes that can be reasoned about and verified independently.

A large refactor is justified only when required by an explicit architectural constraint or when smaller changes would preserve unacceptable coupling or correctness risk.

Do not mix:

- unrelated formatting churn;
- dependency upgrades;
- product changes;
- architecture migrations

into one work unit unless they are causally required.

## 13. Dependency policy

New dependencies require a recorded reason covering:

- problem solved;
- maintenance posture;
- security implications;
- runtime/build impact;
- whether an existing dependency or standard library can satisfy the need.

Dependency additions must be pinned and lockfiles committed.

## 14. Verification levels

The repository should progressively support:

```text
Level 1 — format / lint / type checks
Level 2 — unit + contract tests
Level 3 — integration tests
Level 4 — end-to-end critical journeys
Level 5 — performance / resilience / security checks
```

A change should run the highest practical level relevant to its blast radius.

## 15. CI policy

CI should eventually enforce at least:

- formatting/linting;
- type or compile checks;
- unit tests;
- contract tests;
- integration tests for changed boundaries;
- secret scanning;
- dependency/security checks;
- migration validation where applicable.

Release gating must be defined by risk rather than by a single global test command.

## 16. Observability as verification

Production readiness requires evidence that a feature can be observed after deployment.

Relevant components should expose:

- structured logs;
- metrics;
- traces/correlation IDs;
- failure reasons;
- retry state;
- latency;
- tenant-safe audit records where required.

The canonical `event_id` should connect ingestion, processing, attribution and delivery telemetry without forcing sensitive payloads into logs.

## 17. Branch and commit discipline

Commits should represent coherent milestones.

Recommended pattern:

```text
spec: define behavior
contract: formalize boundary
implementation: add behavior
verification: strengthen coverage
refactor: simplify without behavior change
```

Do not create commits whose only purpose is to hide unfinished verification.

For agent-driven work, smaller coherent commits reduce recovery and review cost.

## 18. No-AI-slop constraints

Agents must not:

- generate boilerplate without a concrete requirement;
- create abstractions for hypothetical future needs;
- copy provider schemas into the domain model;
- add speculative microservices;
- write tests that merely restate implementation details;
- weaken assertions to make tests pass;
- suppress warnings/errors without understanding their cause;
- claim completion without verification evidence.

Useful output is measured by correctness, observability and maintainability, not by code volume.

## 19. Required artifacts before production implementation

Before the first production feature, the repository should contain:

- Product Spec;
- Domain Model;
- Event/Tracking Contracts;
- Attribution Model;
- Architecture;
- Multi-tenancy + RBAC;
- MVP Spec;
- Engineering Harness;
- implementation-level contracts for the first slice;
- test/eval plan for the first slice.

The first implementation slice should be small enough to prove the architecture without prematurely implementing the whole platform.

## 20. Definition of done

A work unit is complete when:

```text
Behavior specified
    +
Contracts updated
    +
Implementation complete
    +
Relevant tests green
    +
Negative/security cases checked
    +
Observability adequate
    +
Documentation synchronized
    +
Diff reviewed
```

No single green test suite substitutes for this complete evidence chain.

## 21. Next engineering step

After this harness, the next artifact is the **first implementation slice specification**: a narrow, end-to-end vertical slice that proves ingestion, canonical events, tenant isolation, persistence, processing and a minimal observable dashboard/debugger path before broader feature expansion.
