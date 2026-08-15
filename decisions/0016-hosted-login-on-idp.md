# ADR-0016: Hosted login is served by `rba-idp` (same origin as the PEP API)

- Status: Accepted
- Date: 2026-08-15
- Relates to [ADR-0013](0013-idp-staged-start.md) IdP-5, [ADR-0014](0014-thesis-scale-idp-platform.md),
  [ADR-0002](0002-backend-tech-stack.md) (React remains for admin).

## Context

IdP-5 is the Auth0/Authentik-style **hosted login page**: applications send users
to the IdP; the user never types a password into the demo app. Options:

1. New `rba-frontend` repo (React + Vite) now, login-only, admin later.
2. Colocate a React SPA under `rba-idp`.
3. **Serve the page from `rba-idp`** (Jinja + static JS) on the same origin as
   `POST /login`.

The catalog already allows `rba-frontend` to “start colocated with idp”. Standing
up a second UI stack/repo before admin exists would be IdP-6 work in disguise.
ADR-0002’s React choice is for the **demo app + admin console**, not a requirement
that Universal Login be an SPA.

## Decision

1. **IdP-5 lives in `rba-idp`.** `GET /` and `GET /login` return HTML. The page
   calls the existing JSON API (`POST /login`, `/mfa/verify`, `/session`,
   `/logout`). Contracts stay 0.2.0.
2. **Same origin, no CORS, no OIDC.** `application_id` is a query param (default
   seeded app). Client IP is injected into the page (`X-Forwarded-For` or peer)
   because the browser cannot supply a trustworthy public IP for `LoginRequest`.
3. **Reasons are first-class UI.** MFA / reauth / block / success screens render
   the PDP `reasons` list — that is the thesis differentiator on the hosted page.
4. **`rba-frontend` + React wait for IdP-6** (admin: users, apps, decisions,
   policy). Do not implement OIDC/SAML/SCIM.

## Consequences

- Positive: one process to run for a demo (`uvicorn` on :8001); JSON clients
  unchanged; login UX is actually “on the IdP”.
- Negative: two UI technologies until admin lands (Jinja login vs React admin).
  Acceptable; extract to `rba-frontend` only if admin needs to share chrome.
- Follow-up: IdP-6 admin console. Do not fold admin HTML into these templates.
