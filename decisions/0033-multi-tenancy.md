# 33. Multi-tenancy: organisations with isolated directories and policies

- Status: Accepted
- Date: 2026-10-21
- Extends: [ADR-0029](0029-rbac-roles-and-permissions.md) (RBAC now lives inside
  a tenant), [ADR-0019](0019-groups-grant-app-access.md).
- Related: [ADR-0034](0034-base-policy-and-engine-mode.md) (policy per tenant),
  [ADR-0036](0036-brand-and-per-tenant-login.md) (branding per tenant).
- **Changes scope:** removes "Arquitectura multi-tenant" from the exclusions in
  E90 §1.2.2. The document must be updated to match.

## Context

E90 excludes multi-tenancy from scope, while chapter 4's business model is a
shared platform where "several organisations share one platform and each keeps
its data isolated". The prototype could not demonstrate the thing the business
model sells.

Today the IdP has one directory (`users.email` unique globally), several
applications each with its own roles, and the PDP holds one policy bundle. A
second demo application on top of that would show per-app branding and roles,
but not isolation.

The payments security architect interviewed for E90 stated a contractual
condition: one customer's events must not be used to train another customer's
model.

## Decision

A **tenant** is an organisation that uses RokuSec. Isolation is logical (a
`tenant_id` on every owned row, enforced server-side), in the existing shared
databases. Not a database per tenant.

### What belongs to a tenant

| Entity | Scope |
|---|---|
| Applications | exactly one tenant |
| Users | one tenant; `email` unique **per tenant**, so the same address can be two people in two organisations |
| Groups, roles, permissions | one tenant (roles stay scoped to one application, ADR-0029) |
| Sessions, MFA challenges, passkeys, auth codes | follow the user |
| Policy bundle (bands, ladder, base policy, engine mode) | one per tenant (ADR-0034) |
| Decisions, audit records, events | carry `tenant_id` |
| Branding of the hosted login | one per tenant (ADR-0036) |
| Behavioural profile (Redis) | keyed by `user_id`, already globally unique and opaque; unchanged |
| Scoring model | **one global model** for now |

### How a login finds its tenant

From the **application that started the flow** (its `client_id`), never from
anything the user types. The login form then looks the email up only inside
that tenant.

### Who sees what

- **Tenant operators** (the E90 *administrador* and *operador de soporte*, the
  "Diego" persona) see only their own tenant in the console. The scoping is
  enforced by the IdP's admin API and by a `tenant_id` filter on the PDP's
  decision reads, not by the UI.
- **Platform operator** (the E90 *operador de infraestructura*, on the provider's
  side): sees all tenants, creates tenants, and switches the console between
  them. A distinct role, not a flag. Platform operators are users of a third,
  system tenant, `rokusec`.

### How an operator reaches the console: by email domain

The console never asks for an organisation; asking would teach every operator
the names of the others. Instead each tenant registers its **operator email
domains** (home realm discovery, as Entra ID and Okta do):

- a domain belongs to exactly one tenant across the platform (unique);
- the console login is email + password; the server maps the domain to the
  tenant, looks the user up only there, and requires a console role;
- an unknown domain, an unknown user, a wrong password and a missing console
  role all return the same `INVALID_CREDENTIALS` with the same timing (the
  bcrypt dummy hash), so the login reveals no tenant's existence;
- customers are unaffected: an application login takes its tenant from the
  application's `client_id`, and a customer's address can be any domain;
- console logins are scored by the engine under their tenant's policy, like any
  other login.

Seeded domains use the reserved `.example` TLD: `rokubank.example`,
`kazelabs.example`, `rokusec.example`.

### Contracts

`tenant_id` is added to `RiskEvaluateRequest`, to the decision record, to
`rba.decision.made.v1`, and the policy endpoints become
`GET`/`PUT /tenants/{tenant_id}/policy`. A minor version bump in
`rba-contracts`; the old single-policy endpoint is removed rather than kept as a
default tenant.

### Policy storage

Policy bundles move from a YAML file inside the PDP container to a
`tenant_policies` table in `rba_decision`. Found while planning this ADR:
`PUT /policy` updated only the in-memory copy and the file of the replica that
served the request, so with the HPA at 2–5 replicas a policy change reached one
pod and was lost on restart (contrary to RNF-05 and RNF-10). Replicas read the
table and cache it for a few seconds.

### Seed

Two tenants: **Roku Bank** (consumer banking) and **Kaze Labs** (a software
company's internal portal for its employees), each with its own users, groups,
roles, operators and policy, plus one platform operator.

## Consequences

**Good.**

- The prototype demonstrates what chapter 4 sells: two organisations on one
  platform, neither able to see the other.
- The scope exclusion and the business model stop contradicting each other.
- Per-tenant policy makes ADR-0034's monitor → active rollout a per-customer
  decision, which is how it would be sold.

**Costs and risks.**

- The largest code change in this phase: IdP schema and every admin query,
  contracts, PDP policy storage and reads, the console, the seed. Every admin
  query is a place where a missing filter leaks data across tenants; each admin
  endpoint gets a cross-tenant test.
- `create_all()` has no migrations; the cloud databases are created fresh on
  every `up`, and k3d is reset, so no migration is written.
- One global model trained on a public dataset is not per-tenant learning. A
  per-tenant model trained only on that tenant's events is future work, and is
  what the security architect's condition would require in production.
- The "customer's own cloud" modality of chapter 4 is a single-tenant install of
  the same software and is not built.
