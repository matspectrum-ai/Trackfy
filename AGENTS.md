# Trackfy Engineering Governance

## Purpose

Trackfy is built specification-first. Production implementation must follow the approved product, domain, tracking, attribution, architecture and multi-tenancy specifications in `docs/`.

## Authority order

When requirements conflict, use this order:

1. Explicit current task requirements.
2. Approved product/domain/contract specifications.
3. Architectural decisions recorded in `docs/architecture/` and ADRs.
4. Tests and executable contracts that encode approved behavior.
5. Existing implementation.

Existing code is evidence, not authority, when it conflicts with an approved specification.

## Engineering loop

Every non-trivial change follows:

```text
Observe → Specify → Plan → Act → Verify → Iterate
```

Do not begin implementation when a required behavior is materially ambiguous. Record the ambiguity as an explicit decision or deferred item instead.

## Specification discipline

Before implementation:

- identify the governing specification;
- define observable behavior;
- define acceptance criteria;
- define contracts at integration boundaries;
- identify tenant/security implications;
- identify failure modes;
- define the test strategy.

Do not invent product behavior merely to make implementation easier.

## Testing discipline

Tests must describe externally observable behavior and important invariants.

Prefer:

- contract tests for public boundaries;
- unit tests for deterministic domain logic;
- integration tests for persistence and infrastructure boundaries;
- end-to-end tests for critical user journeys;
- property/invariant tests where state-space complexity justifies them.

A passing test suite does not prove compliance if the tests encode the wrong behavior. Tests must remain traceable to specifications.

## Contract discipline

Public or cross-component contracts must be versioned explicitly.

Breaking changes require a new contract version or an approved migration strategy.

Provider-specific schemas belong in integration adapters. They must not leak into the canonical Trackfy domain model without an explicit architectural decision.

## Multi-tenancy

Tenant isolation is a correctness requirement, not a UI feature.

Every tenant-owned operation must establish:

```text
User → Membership → Organization → Workspace → Resource
```

Never trust tenant IDs supplied by a client as authorization evidence.

Authorization must be enforced server-side at every protected boundary, including APIs, workers, analytics, exports, debugger data and integration delivery.

## Tracking data

Canonical events are immutable evidence.

Derived sessions, attribution results, lifecycle projections, aggregates and dashboards must be reproducible from canonical facts and configuration.

Never mutate canonical event history to fix a dashboard result. Correct the derived projection or its transformation rules.

## Financial data

Never collapse gross revenue, approved revenue, pending revenue, refunds, fees, commissions, costs and profit into one mutable amount.

Financial lifecycle facts and their source provenance must remain distinguishable from derived reporting metrics.

## Agent behavior

Coding agents must:

- inspect relevant specifications before editing;
- minimize unrelated changes;
- prefer the smallest coherent change;
- avoid speculative abstractions;
- preserve existing contracts unless the task changes them deliberately;
- run the strongest practical verification after changes;
- report failures and unresolved uncertainty instead of masking them.

Agents must not:

- rewrite specifications to fit existing code without explicit authorization;
- remove tests merely because they fail after a change;
- add dependencies without documenting the reason;
- expose secrets or credentials;
- bypass authorization checks to unblock development;
- silently introduce product behavior not defined by the governing specification.

## Definition of done

A change is not complete until:

1. the governing specification is satisfied;
2. relevant contracts are updated when behavior changes;
3. tests cover the acceptance criteria;
4. verification has been executed and results inspected;
5. security and tenant implications have been reviewed;
6. documentation is updated where the behavior is user/operator significant;
7. the diff is limited to the intended change.

## Documentation map

- `docs/product/` — product behavior and scope.
- `docs/attribution/` — attribution semantics.
- `docs/contracts/` — event and integration contracts.
- `docs/architecture/` — system boundaries and technology direction.
- `docs/security/` — tenancy, RBAC and security constraints.
- `docs/mvp/` — MVP scope and release criteria.
- `docs/engineering/` — development harness, verification and agent workflow.

## Change policy

Changes to product semantics, domain concepts, public contracts, security boundaries or architectural invariants require a documentation change in the same work unit before implementation proceeds.

Commit messages should explain the intent of the change and use a consistent conventional-commit style.
