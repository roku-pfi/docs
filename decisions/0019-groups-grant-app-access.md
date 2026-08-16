# ADR-0019: Groups grant app access on the IdP; admin stays `is_admin`

- Status: Accepted
- Date: 2026-08-15
- Relates to [ADR-0013](0013-idp-staged-start.md) IdP-7,
  [ADR-0012](0012-thin-idp-end-product.md) (groups as stretch),
  [ADR-0017](0017-admin-console-colocated-on-idp.md) (`is_admin` for the console)

## Context

IdP-7 is the documented stretch: **groups / app-scoped permissions** for demo
clients — who may sign in to which registered application. It is not an
enterprise authorization engine (no OIDC/SAML/SCIM, no org units, no
fine-grained RBAC).

Admin console auth already uses an `is_admin` flag (ADR-0017). Replacing that
with groups in the same slice would mix operator capability with app access
and reopen the “do everything now” failure mode.

Login currently accepts any enabled user for any enabled application, then
asks the PDP. Without a grant check, groups would be directory labels only.

Options:

1. Full RBAC / policy engine (Authentik-shaped roles, per-endpoint perms).
2. Groups as labels in admin only (no login effect).
3. **Groups + memberships + app-scoped `access` grants that gate login
   before the PDP** (chosen). `is_admin` stays the operator flag.

## Decision

1. **Tables live in `rba_idp`:** `groups`, `group_memberships`,
   `group_app_grants`. Not in the PDP, not a new service.
2. **After password verify, before `/risk/evaluate`:** the user must belong
   to a group that grants `permission=access` on the requested application.
   MFA completion re-checks the same grant before minting a session.
3. **`ACCESS_DENIED` is an IdP login outcome** (HTTP 200, like
   `INVALID_CREDENTIALS`). It is **not** a PDP `BLOCK` and does **not** call
   the scorer. Hosted login shows a distinct panel.
4. **`is_admin` still gates `/admin/api/*`.** Groups do not replace the
   operator flag. Seeded `grp_operators` grants the admin user `access` to
   `idp-admin-console`; the demo user has no such grant.
5. **One permission value:** `access`. Enough to demo app-scoped access;
   not a permission matrix.
6. **Seed so existing demos keep working:** `grp_banking` (demo user →
   banking app) and `grp_operators` (admin → admin console and banking).
   New users have no grants until an operator assigns them.

## Consequences

- Positive: Authentik/Auth0-shaped “who can use this app”; RBA still only
  scores logins that are authorized; contracts stay additive (`rba-contracts`
  0.4.0); evaluate latency path unchanged.
- Negative: new users cannot log in until grouped (operator workflow);
  existing sessions are not revoked when a grant is removed (TTL).
- Follow-up: k8s/Helm/observability. Still no OIDC/SAML/SCIM.
