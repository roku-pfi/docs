# ADR-0025: Demo app lives in another namespace and authenticates via the IdP

- Status: Accepted
- Date: 2026-08-16
- Amends [ADR-0024](0024-separate-demo-app.md) (where the client is deployed)
  and [ADR-0020](0020-local-k8s-k3d-helm.md) (Ingress was IdP-only in `rba`).
  Does **not** change PDP/PEP: the app still does not call Freeman
  ([ADR-0012](0012-thin-idp-end-product.md),
  [ADR-0015](0015-rba-is-the-thesis-core.md)).

## Context

The RBA platform today is one Helm release in namespace **`rba`** (IdP, PDP,
workers, data plane). ADR-0024 made Demo-3 a separate repo (`rba-demo-banking`)
but did not say where it runs.

The demo that matches the product: a **tenant application** in **another
namespace** uses the RBA platform the way a real bank would use Auth0 — send
the user to login, get a session back. Putting the bank in `rba` next to the
PDP still looks like one system. Letting the bank `POST` to
`decision-service.rba.svc` would skip the PEP and leak the risk API.

Options considered:

- Deploy the demo app in `rba` (rejected — same tenancy as the platform).
- App in another namespace calls `/risk/evaluate` in-cluster (rejected —
  that *is* embedding the PDP in the bank).
- **App in another namespace; browser redirects to the IdP Ingress** (chosen).

## Decision

1. **Two namespaces.** Platform stays `rba`. Demo-3 (`rba-demo-banking`)
   deploys in **`demo`**. Separate Helm release (chart can live in
   `rba-infra` or the app repo). Compose may still run both on localhost
   ports for the no-cluster loop.

2. **How the app “uses RBA”:** it **authenticates through `rba-idp`**. The
   user hits the bank → redirect to hosted login (IdP Ingress) → IdP verifies
   identity, asks the PDP, enforces ALLOW/MFA/REAUTH/BLOCK → redirect back to
   the bank with the thin `redirect_uri` code (ADR-0024). RBA is the IdP’s
   login decision, not a library inside the bank.

3. **No in-cluster PDP from `demo`.** The banking pods must not call
   `decision-service`. Cross-namespace ClusterIP to the PDP is out of the
   demo contract. (A NetworkPolicy that denies it is a nice later proof, not
   a Demo-3 blocker.)

4. **Ingress for both.** IdP remains the platform entry (today `:8080`). The
   bank gets its own host/path (e.g. `demo.localhost` or a second path on
   Traefik). Browser, not mesh, is the integration.

5. **Still out:** OIDC/SAML suite, the bank as a second IdP, GitOps as a
   blocker.

## Consequences

- Positive: the defense shows a tenant app and a platform in different
  namespaces — the same story as “our service vs their app.”
- Negative: two Ingresses, DNS/hosts, and a callback across origins; local
  demo is slightly heavier than one namespace.
- Follow-up: Demo-3 creates namespace `demo` + Deployment + Ingress and
  registers `redirect_uri`. Demo-2 (seed) still comes first.
