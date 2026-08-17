# ADR-0024: Demo client is a separate app that uses the RBA platform

- Status: Accepted (deploy namespace amended by
  [ADR-0025](0025-demo-app-separate-namespace.md))
- Date: 2026-08-16
- Supersedes the Demo-3 **colocation** choice in
  [ADR-0022](0022-product-demo-over-gitops.md) (“banking UI on `rba-idp` `/app`”;
  “no new repo/service”). Does **not** change: PDP/PEP split, “the app never
  calls Freeman”, or end-user opacity
  ([ADR-0012](0012-thin-idp-end-product.md),
  [ADR-0015](0015-rba-is-the-thesis-core.md),
  [ADR-0023](0023-end-user-login-is-opaque.md)).

## Context

Horizon D was meant to show **an application that uses this RBA system** — a
person logs into a bank; the IdP asks the PDP; friction appears only when the
login looks unlike that user. ADR-0022 then colocated a mocked banking page on
the IdP (`GET /app`) to avoid a new microservice.

That collapses two products into one origin. The IdP *is* the RBA platform
(Authentik/Auth0-shaped). The bank is a **client of that platform**. A `/app`
route on `rba-idp` looks like the IdP *is* the bank, which is the wrong demo.

A separate origin also needs a return path (the IdP session today lives on the
IdP host). Full OIDC/SAML is still out of thesis scope (ADR-0014).

Options considered:

- Keep `/app` on `rba-idp` (rejected — mocked client, wrong product split).
- New app that calls `POST /risk/evaluate` itself (rejected — bypasses the PEP;
  the thesis is IdP-enforced RBA, not a raw risk SDK in the bank).
- **Separate demo app repo that redirects to the IdP** (chosen).

## Decision

1. **Demo-3 is `rba-demo-banking`**, its own polyrepo service (own image, own
   origin). It is the registered client `demo-banking-app`. Dummy balances /
   home only — **no** score, action, or reasons (ADR-0023).

2. **It uses RBA; it does not implement the scorer.** Sign-in redirects to
   hosted login on `rba-idp`. The IdP verifies password, asks the PDP, enforces
   `ALLOW` / `MFA` / `REAUTH` / `BLOCK`, then returns the user to the app.
   Freeman stays on `rba-decision-service`.

3. **Return path is a thin redirect, not OIDC.** Register a `redirect_uri` on
   the IdP application. After `AUTHENTICATED`, the IdP redirects there with a
   one-time code (or equivalent) that the app exchanges for the existing IdP
   session token. No discovery document, no SAML, no OAuth provider suite.

4. **Optional `demo-forum-app`** remains a second *policy* (looser bands), not
   a second repo unless the walkthrough needs two UIs.

5. **Still out:** calling Freeman from the bank; identity tables in the PDP;
   colocation of the customer UI on `rba-idp`; GitOps (K8s-3) as a blocker.

## Consequences

- Positive: the defense shows a real relying party and a real IdP, which is
  how RBA is sold. Opacity and the PDP/PEP split stay intact.
- Negative: one more repo, Ingress, and a small callback contract. Cross-origin
  session is more work than `/app` on the same host.
- Follow-up: Demo-2 (seed + scenarios) still comes first. Demo-3 creates
  `roku-pfi/rba-demo-banking`, adds `redirect_uri` on applications, and a Helm
  Deployment. Do not skip Demo-2 to chase the UI.
