# ADR-0014: Thesis-scale IdP platform (Authentik/Auth0-shaped, not enterprise)

- Status: Accepted
- Date: 2026-08-15
- Amends product metaphor in [ADR-0012](0012-thin-idp-end-product.md);
  sequencing still [ADR-0013](0013-idp-staged-start.md). PDP/PEP split unchanged.

## Context

“Thin IdP” was easy to read as a JSON login wrapper in front of the PDP. The
intended product is closer to **Authentik or Auth0**: a platform that applications
hand login to, with users, registered clients, a hosted login, sessions, MFA, and
an admin console — plus **explainable RBA** as the differentiator (their adaptive
MFA is opaque).

That must stay **thesis-sized**. Carrier-grade / enterprise IdP means federation,
SCIM, LDAP/AD, social connections, full OIDC/SAML suites, multi-tenant HA, and
authorization engines. Building those would swallow the project.

## Decision

1. **Product metaphor:** a small **identity platform**. Demo apps are *clients of
   the IdP*, not callers of a raw risk API. Users log in on the IdP; the admin
   console is how you operate it (users, applications, login/decisions, policy).
2. **What “like Auth0/Authentik” means here**

   | In (thesis) | Out (enterprise) |
   |---|---|
   | Users + password store | LDAP/AD, social login, passkeys (unless leftover time) |
   | Registered applications (clients) | App marketplace, thousands of tenants |
   | Hosted login + session | Full OIDC discovery, SAML, OAuth device-flow, SCIM |
   | MFA challenge (mock OTP fine) | WebAuthn/TOTP provider matrix |
   | Risk via existing PDP + reasons | Opaque “anomaly detection” black box |
   | Admin: users, apps, decisions, policy | Fine-grained authz engines, org units, billing |

3. **Protocol:** hosted login page + IdP session (cookie/token) is enough. Do **not**
   implement an OIDC/SAML IdP in IdP-2…5. A minimal auth-code flow for one demo
   app is a later stretch, never a blocker for login + enforce + session.
4. **Architecture unchanged:** `rba-idp` = PEP / platform shell; `decision-service`
   = PDP. Applications do not call Freeman. No identity tables in `rba_decision`.
5. **Stages:** IdP-2 seeds **users and at least one registered application**
   (Auth0 “application” / Authentik “provider+application”, simplified). Admin
   console stays IdP-6.

## Consequences

- Positive: end product matches “RBA as a login platform”; grading still has a
  clean PDP; scope has a visible cut line against Authentik/Auth0 feature lists.
- Negative: people will ask for OIDC — answer is ADR-0014, not a protocol sprint.
- Follow-up: `status.md` / plan §1.1 and Phase 7 wording; IdP-2 implements users +
  applications, not OIDC.
