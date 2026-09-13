# Trackfy — Multi-tenancy & RBAC v0.1

**Status:** Draft  
**Version:** 0.1  
**Scope:** Tenant isolation, Organization/Workspace boundaries, memberships, roles, permissions, API keys, service identities and authorization invariants.

This document defines the logical authorization model. It is not a database schema, Supabase RLS implementation or final API specification.

## 1. Objective

Trackfy is a multi-tenant SaaS. Tenant isolation is a core product invariant, not an implementation detail.

The authorization model must answer deterministically:

1. Who is acting?
2. Which Organization is the request operating within?
3. Which Workspace and Site are in scope?
4. Which role and permissions apply?
5. Is the requested resource visible to that principal?
6. Is the action allowed on that resource?
7. Is the credential itself permitted to perform the operation?

UI visibility must never be treated as authorization.

## 2. Tenant hierarchy

```text
User
  │
  ├── Membership ──→ Organization
  │                    │
  │                    ├── Workspace
  │                    │     ├── Site
  │                    │     ├── Tracking data
  │                    │     ├── Integrations
  │                    │     └── Reports / Rules
  │                    │
  │                    ├── Members
  │                    └── Organization-level configuration
  │
  └── Credentials / sessions
```

Organization is the primary collaborative tenant boundary.

Workspace is the operational/reporting isolation boundary inside an Organization.

Site is the tracked property boundary and may be the origin of browser-side collection.

## 3. Principal types

### 3.1 User

A human authenticated account. A User has no implicit access to tenant data merely because the account exists.

Access is granted through explicit membership.

### 3.2 Organization Membership

A relationship between User and Organization.

Conceptual attributes:

- organization;
- user;
- role;
- status;
- granted scope where supported;
- timestamps;
- audit metadata.

Initial membership states:

`invited`, `active`, `suspended`, `removed`.

A non-active membership must not authorize normal tenant operations.

### 3.3 API Key

A programmatic credential used by a human-controlled or application-controlled client.

An API key must have:

- explicit tenant scope;
- explicit resource scope where applicable;
- declared capability/scope set;
- status;
- creation/last-used metadata;
- revocation semantics.

The secret value is shown only at creation time where product semantics permit and must not be stored or returned in plaintext through normal management APIs.

### 3.4 Integration Identity

A machine identity associated with an external integration or server-side producer.

Integration identities are not equivalent to human Users and should not inherit arbitrary human roles.

## 4. Organization boundary

Organization owns or scopes:

- memberships;
- Workspaces;
- organization-level settings;
- integration definitions;
- organization-level credentials and policies;
- billing/entitlement state when introduced.

An Organization must never be inferred from an untrusted browser field alone.

The authenticated principal and credential context must resolve the effective Organization scope.

## 5. Workspace boundary

A Workspace provides an operational context within an Organization.

Workspace-scoped resources include, conceptually:

- Sites;
- tracking configurations;
- campaigns/source metadata;
- ad-account connections;
- sales integrations;
- webhook endpoints;
- API keys with Workspace scope;
- dashboards/reports;
- rules/alerts;
- tracking and analytical data.

Default rule:

```text
Organization access ≠ automatic unrestricted Workspace access
```

The initial product may grant organization roles broad Workspace visibility, but the authorization model must be capable of restricting access explicitly.

## 6. Site boundary

A Site belongs to a Workspace unless a later product decision explicitly introduces sharing.

Site-scoped browser ingestion credentials may publish events only for their authorized Site(s).

A Site credential must not grant dashboard read access merely because it can submit tracking data.

## 7. Roles

Initial human roles:

| Role | Intent |
|---|---|
| Owner | Full Organization control, including destructive/admin actions and ownership lifecycle |
| Admin | Administrative management without ownership transfer authority unless explicitly granted |
| Analyst | Read/analysis access to permitted data and reporting |
| Media Buyer | Campaign, acquisition and performance operations |
| Viewer | Read-only access to permitted dashboards and reports |
| Developer | Tracking, API, integration and technical configuration access |

Roles are permission bundles, not security boundaries by themselves.

## 8. Permission model

Permissions should be explicit capability identifiers rather than hard-coded role checks throughout application code.

Conceptual permission families:

```text
organization.read
organization.manage
organization.members.manage
organization.billing.manage

workspace.read
workspace.manage
workspace.delete

site.read
site.manage
site.tracking.manage

tracking.read
tracking.debug
tracking.export

campaigns.read
campaigns.manage

integrations.read
integrations.manage
integrations.credentials.manage

api_keys.read
api_keys.manage

webhooks.read
webhooks.manage

orders.read
orders.manage

attribution.read
attribution.manage

analytics.read
reports.read
reports.manage

rules.read
rules.manage

security.audit.read
```

The final permission catalog belongs to the RBAC implementation specification.

## 9. Effective authorization

Authorization should evaluate a request using:

```text
Principal
 + Membership / machine identity
 + Organization scope
 + Workspace scope
 + Resource ownership
 + Permission
 + Resource state
 + Credential scope
 → Allow / Deny
```

A request must fail closed when the effective scope cannot be determined.

## 10. Resource authorization

Every tenant-owned resource must have a deterministic ownership/scope path.

Example:

```text
Event
 ↓
Site
 ↓
Workspace
 ↓
Organization
```

A user authorized for Organization A must never retrieve an Event belonging to Organization B merely by changing an ID in a request.

This applies equally to direct resource reads, filters, exports, batch endpoints, background jobs and debugger timelines.

## 11. API authorization

Application APIs must derive authorization server-side.

A typical request context is:

```text
Authenticated principal
       ↓
Resolve membership / credential
       ↓
Resolve effective org/workspace scope
       ↓
Check permission
       ↓
Check resource ownership
       ↓
Execute operation
```

Client-provided `organization_id`, `workspace_id` or similar values are selectors, not proof of authorization.

## 12. Browser tracking credentials

Browser collection requires a publishable/site-scoped credential or equivalent identifier.

It may permit event submission but must not expose privileged capabilities such as:

- organization administration;
- reading arbitrary tracking data;
- managing integrations;
- retrieving API secrets;
- modifying RBAC.

Browser credentials should be revocable and rotatable.

## 13. API key scopes

API keys should support least-privilege scopes such as:

```text
tracking:write
tracking:read
orders:read
orders:write
analytics:read
integrations:read
integrations:write
webhooks:write
```

An API key with `tracking:write` must not implicitly gain `organization.manage`.

Workspace and Site restrictions should be supported where operationally useful.

## 14. Integration authorization

External integration credentials must be isolated from ordinary user credentials.

```text
Human User
   ≠
Trackfy API Key
   ≠
Provider Integration Credential
```

A user may be allowed to configure an integration without being allowed to retrieve its secret material.

Secret access should be narrower than metadata/configuration access.

## 15. Background jobs

Authorization must remain valid for asynchronous work.

A job should carry enough scoped context to enforce:

- Organization;
- Workspace;
- initiating identity or system actor where relevant;
- resource identifiers;
- required capability.

Workers must not bypass tenant checks merely because they are trusted infrastructure.

## 16. Analytics authorization

Analytics access must enforce tenant and Workspace scope before query execution.

A filter applied only in the UI is insufficient.

Aggregates must not allow cross-tenant leakage through:

- grouped results;
- exports;
- comparison queries;
- cohort reports;
- debugger traces;
- cached results.

Read models must retain enough scope metadata to enforce authorization safely.

## 17. Event Debugger authorization

Raw event evidence is more sensitive than ordinary aggregate metrics.

Debugger access should require a dedicated capability such as `tracking.debug`.

A Viewer may be allowed to inspect aggregate reports while being denied raw event payloads.

Raw payload exposure should also respect data minimization/privacy rules.

## 18. Data isolation invariants

1. No tenant-owned object exists without a resolvable authorization scope.
2. Every read/write path enforces that scope server-side.
3. Background workers enforce the same scope as synchronous requests.
4. Exports are subject to the same authorization rules as interactive queries.
5. Caches must be tenant-aware.
6. Logs and diagnostics containing customer data must preserve appropriate access boundaries.
7. Cross-Organization references are invalid unless an explicit future sharing mechanism defines them.
8. Site ingestion cannot write into another Site by modifying a request field.

## 19. Ownership and deletion

Owner-level operations may include:

- transferring ownership;
- removing members;
- deleting Organization resources;
- rotating/revoking critical credentials.

Destructive actions should be explicitly permissioned and auditable.

Deletion semantics for event evidence and derived analytics are deferred to the retention/privacy specification.

## 20. Auditability

Security-sensitive actions should generate audit records, including where applicable:

- member invitation/removal/role change;
- API key creation/revocation;
- integration credential change;
- tracking configuration changes;
- RBAC changes;
- destructive resource operations;
- ownership transfer.

Audit records are separate from marketing tracking events.

## 21. Session and authentication boundary

Authentication establishes who the principal is. Authorization establishes what that principal may access.

A valid authenticated session must not by itself authorize access to any Organization.

Session expiration/revocation semantics belong to the authentication specification.

## 22. Deny-by-default behavior

When any required authorization input is absent or contradictory, the default result is deny.

Examples:

```text
No membership       → deny
Membership removed  → deny
Wrong workspace     → deny
Missing API scope   → deny
Unknown resource    → deny
Ambiguous ownership → deny
```

Error responses should avoid leaking whether unauthorized resources exist.

## 23. Role intent matrix

Initial qualitative matrix:

| Capability family | Owner | Admin | Analyst | Media Buyer | Viewer | Developer |
|---|---:|---:|---:|---:|---:|---:|
| Organization administration | full | manage | no | no | no | limited |
| Member/RBAC management | full | manage | no | no | no | no |
| Workspace management | full | full | read | read/manage acquisition | read | manage |
| Tracking/debugger | full | full | read | read | limited | full |
| Analytics/reports | full | full | full | performance | read | read |
| Campaign operations | full | full | read | full | read | read |
| Integrations | full | full | read | read | no | full |
| API keys | full | manage | no | no | no | manage |
| Orders/lifecycle | full | full | read | read | read | read |
| Attribution configuration | full | manage | read | manage | read | manage |

This matrix is intentionally qualitative and must not be treated as the final machine-readable permission set.

## 24. Acceptance criteria

### AC-01 — Organization isolation
A User with membership only in Organization A cannot read, mutate, export or debug resources owned by Organization B.

### AC-02 — Workspace isolation
When a Workspace-scoped permission is required, a User without that Workspace scope cannot access its resources even when they belong to the same Organization.

### AC-03 — Credential least privilege
A browser/site credential capable of tracking writes cannot retrieve privileged tenant data or administrative secrets.

### AC-04 — API key scope
An API key with `tracking:write` cannot perform operations requiring `organization.manage`.

### AC-05 — Background isolation
A background job created for Organization A cannot read or mutate Organization B resources by changing identifiers.

### AC-06 — Debugger protection
A principal without `tracking.debug` cannot retrieve raw event traces merely because aggregate analytics access is available.

### AC-07 — Deny by default
Requests with missing/invalid/ambiguous authorization context are denied without revealing unauthorized resource existence.

### AC-08 — Auditability
Security-sensitive authorization changes generate auditable records with actor and tenant context.

## 25. Deferred decisions

- exact permission catalog;
- exact role-to-permission mapping;
- whether Workspace membership is independent from Organization membership;
- whether resources can be shared across Workspaces;
- service-account semantics;
- session authentication provider;
- API key format and hashing algorithm;
- key rotation and expiry policy;
- fine-grained attribute-based policies beyond RBAC;
- retention/deletion of audit records;
- privacy/compliance requirements.

## 26. Next step

The next specification should be **MVP Spec v0.1**, using the Product, Domain, Event/Tracking, Attribution, Architecture and Multi-tenancy/RBAC decisions to define the smallest complete product that already delivers real tracking-to-profit value.