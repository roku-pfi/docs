# ADR-0015: RBA is the thesis core; the IdP is how it is delivered

- Status: Accepted
- Date: 2026-08-15
- Amends emphasis in [ADR-0014](0014-thesis-scale-idp-platform.md) (platform
  shape unchanged). Architecture still [ADR-0012](0012-thin-idp-end-product.md).

## Context

ADR-0014 set the product shell to Authentik/Auth0-shaped. That can be misread as
“the thesis is a mini-Auth0.” The thesis is **risk-based authentication**:
transparent, per-signal-explained, tunable login risk (`ALLOW` / `MFA` /
`REAUTH` / `BLOCK`) on a real request path — the gap vs Okta/Entra/Auth0/Ping.

The IdP exists so that RBA is experienced as a login platform, not as a curl
to `/risk/evaluate`. Without the PDP, features, policy, and explanations, the
IdP is an empty shell and the thesis has no core.

## Decision

1. **Thesis core = RBA.** Grade-critical: `rba-features` parity, Freeman (or
   successor) + policy, `/risk/evaluate`, reasons, leakage result, train/serve
   identity. The IdP must **call** that core, never replace or hide it.
2. **Product shell = thesis-scale IdP** (ADR-0014). Users, apps, hosted login,
   session, MFA, admin — so a client app actually logs in through RBA.
3. **If time is tight:** cut IdP chrome (admin polish, groups, protocol) before
   cutting the risk path, explanations, or k8s/HPA. An IdP that does not show
   risk reasons has missed the thesis.
4. **Admin must surface RBA:** decisions, scores, reasons, policy thresholds —
   not only a user directory.

## Consequences

- Positive: Auth0-like UX without losing the research claim; cut order is
  explicit if calendar slips.
- Negative: every IdP stage after IdP-2 must keep a visible path to the PDP
  (IdP-3 is when RBA re-enters the login flow — do not skip it to chase UI).
