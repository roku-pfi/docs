# ADR-0017: Admin console colocated on `rba-idp`; control-plane reads stay with owners

- Status: Accepted
- Date: 2026-08-15
- Relates to [ADR-0013](0013-idp-staged-start.md) IdP-6, [ADR-0015](0015-rba-is-the-thesis-core.md)
  (admin must show RBA), [ADR-0016](0016-hosted-login-on-idp.md) (React for admin;
  do not fold admin into login templates), [ADR-0002](0002-backend-tech-stack.md).

## Context

IdP-6 is the Authentik/Auth0-style **admin console**: users, registered applications,
a **decision browser** (score / action / reasons), and **policy** thresholds.
Catalog items `rba-admin-api` and `rba-frontend` exist, but standing up two new
repos plus a third UI origin in one stage would repeat the “do everything now”
failure mode ADR-0013 avoided.

Data ownership is already split (database-per-service):

- Users and applications live in `rba_idp`.
- Scored logins with full reasons live in `rba_audit` (audit-service).
- Policy YAML is loaded by `rba-decision-service`.

The IdP must not query those other databases. Admin also must not sit on the
PDP latency path as a shared table.

Options:

1. New `rba-admin-api` + `rba-frontend` (catalog shape).
2. Admin HTML folded into IdP-5 Jinja templates.
3. **Colocate** a React SPA + BFF on `rba-idp`; add HTTP read/write on the
   services that own the data (GET/PUT `/policy` on the PDP, GET `/decisions`
   on audit-service).

## Decision

1. **IdP-6 lives in `rba-idp`.** `GET /admin` serves a React (Vite) SPA.
   `/admin/api/*` is the BFF. Hosted login templates stay login-only
   ([ADR-0016](0016-hosted-login-on-idp.md)).
2. **No `rba-admin-api` / `rba-frontend` repo this stage.** Catalog extract is
   allowed later if admin needs its own deploy or shared chrome with a demo app.
3. **Owners keep their data.** The BFF calls `GET`/`PUT /policy` on
   `rba-decision-service` and `GET /decisions` on `rba-audit-service`. Those
   endpoints are additive control-plane APIs, not evaluate-path hops.
4. **Admin auth is an `is_admin` flag**, not groups (IdP-7). Seeded operator:
   `admin@example.com` / `admin-password`, application `idp-admin-console`.
   Operators sign in through the hosted login (PDP still scores that login).
5. **Fail visible.** Audit or PDP down → HTTP 503 on the corresponding admin
   routes. No fabricated decisions or policy.

## Consequences

- Positive: one `uvicorn` still demos login + admin; RBA reasons are first-class
  in the console; database-per-service holds; evaluate latency path unchanged.
- Negative: `rba-idp` now depends on audit HTTP as well as the PDP; two UI
  stacks remain (Jinja login, React admin) until a later extract.
- Follow-up: IdP-7 groups/permissions. Optional extract to `rba-frontend` /
  `rba-admin-api`. Do not implement OIDC/SAML/SCIM.
