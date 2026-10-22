# 39. Applications are named in URLs by an opaque public client ID

- Status: Accepted
- Date: 2026-10-22
- Related: [ADR-0016](0016-hosted-login-on-idp.md), [ADR-0024](0024-separate-demo-app.md),
  [ADR-0033](0033-multi-tenancy.md).

## Context

Login URLs carried the internal application id: `/login?application_id=roku-bank-web`.
That id is not a secret, and an opaque one would not be either: a client id is
public by design, and what protects a login is the registered redirect URI, the
password, the passkey and the engine. But the readable id told anyone how a
tenant names its applications, made a sibling application easy to guess
(`acme-web`, `acme-admin`), and could never change, since roles, permissions,
decisions and policies all refer to it.

## Decision

Every application has a second, public identifier, `client_id`, of the form
`rk_` + 16 random base62 characters (`rk_Bk7mQ2xVn9PwR4tL`), as Auth0 and Okta do.

- The hosted login takes `?client_id=` only; an internal id in the URL names
  nothing and shows "Aplicación desconocida".
- `POST /login` takes exactly one of `client_id` (browsers, applications) or
  `application_id` (server-side callers); the IdP resolves the former first.
  The hosted page never sees the internal id.
- The internal `application_id` stays the key everywhere else, unchanged.
- New applications get a random client id; the seeded demo applications have
  fixed ones (`rba_idp.client_ids.SEEDED`) so their configuration can name them.
  Applications from before the change are backfilled at init.
- The console shows each application's client id with a copy button.

Contracts 0.14.0.

## Consequences

- URLs no longer reveal or let anyone guess application names.
- A client id could be rotated without touching roles or history (not built).
- One more lookup per login (indexed).
