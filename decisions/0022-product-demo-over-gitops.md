# ADR-0022: Product demo over further GitOps; real step-up + travel rule

- Status: Accepted (Demo-3 colocated `/app` superseded by
  [ADR-0024](0024-separate-demo-app.md); Demo-3 UX amended by
  [ADR-0023](0023-end-user-login-is-opaque.md))
- Date: 2026-08-16
- Amends sequencing in [ADR-0013](0013-idp-staged-start.md) / [ADR-0020](0020-local-k8s-k3d-helm.md)
  follow-ups (K8s-3 deferred). Amends [ADR-0014](0014-thesis-scale-idp-platform.md)
  MFA cut line (mock OTP is no longer enough for the demo). Does **not** change
  the PDP/PEP split ([ADR-0012](0012-thin-idp-end-product.md),
  [ADR-0015](0015-rba-is-the-thesis-core.md)).

## Context

K8s-1 (k3d/Helm) and K8s-2 (Prometheus/Grafana + HPA) already make the
architecture claim runnable. The leftover ops stage (GitOps / reusable CI /
Tilt) does not show **what the product is**: explainable login risk on a real
app, with step-up when the model says so.

The hosted login still uses mock OTP `000000`, does not send `country`, and
has no client app behind `demo-banking-app`. New users look uniformly risky.
Impossible travel was excluded from Freeman (synthetic geo, missing city);
the demo still wants a **physics** check (too far, too fast) without claiming
GPS accuracy. Many users sit on VPNs; treating a VPN egress as a city creates
false teleports.

Options considered:

- Continue K8s-3 (GitOps/CI) as current focus.
- Build a feature-rich IAM (OIDC, TOTP+WebAuthn+SMS, city GeoIP, GBM sidecar).
- **Thin product demo** on the existing IdP + PDP (chosen).

## Decision

1. **Current focus is Demo-1…4**, not K8s-3. Architecture work is enough for
   the thesis ops story until the demo exists. GitOps/CI/Tilt stay later.
   Federation / SCIM / full OIDC stay out.

2. **The model still decides.** Freeman on `rba-decision-service` produces
   `risk_score` + per-signal reasons; policy maps score → `ALLOW` / `REQUIRE_MFA`
   / `REAUTHENTICATE` / `BLOCK`. The demo **app never calls Freeman**. The IdP
   is the PEP; it verifies identity, asks the PDP, and enforces. That *is*
   “AI decides whether this login is accepted.”

3. **Demo stages** (one at a time; do not skip Demo-1 to chase passkeys):

   | Stage | What ships | Explicitly not |
   |---|---|---|
   | **Demo-1** Signals + travel rule | Country on the login path; country-centroid `impossible_travel` in `rba-features`; PDP **escalates** (default ALLOW → MFA, not BLOCK); VPN/hosting ASN **skips** teleport and emits `vpn_or_hosting`. Parity tests green. | GPS, city GeoIP, Freeman categorical for travel, MaxMind product |
   | **Demo-2** Scenarios + seed | Seed `demo@example.com` with a usual home profile; a demo-only control to pick the **next** login context (home / new country / impossible travel / VPN). | Phase 6 attack generator, extra worker |
   | **Demo-3** Client app | A small banking UI **colocated on `rba-idp`** (`/app`, same origin as login). Redirect to hosted login; after `AUTHENTICATED`, land in the app. Show this session’s action + top reasons. Reuse existing `demo-forum-app` policy for “same user, looser app.” | New repo, new microservice, OIDC/OAuth suite |
   | **Demo-4** Real step-up | WebAuthn passkey for `REQUIRE_MFA` (platform authenticator = phone/laptop biometrics). Mock OTP stays for tests. Reasons stay on the challenge screen. Completing MFA does **not** re-score. | TOTP/SMS/push unless WebAuthn is blocked on the defense machine |

4. **Impossible travel** is a deterministic **hard override** computed in
   `rba-features` as `f(event, profile)`, not a Freeman input and not GPS.
   Use a static ISO-country centroid table (same offline and online). Store
   `last_login_country` / `last_login_ts` from **successful** logins. Fire only
   when both countries are present and implied speed exceeds ~1000 km/h.
   Missing country never fires. VPN/hosting ASNs skip the physics check.

5. **Keep the demo small.** In for the walkthrough: seeded home context;
   three scripted logins (home ALLOW, novel country → MFA from Freeman+policy,
   teleport → MFA from the rule); VPN as “untrusted network” not teleport;
   admin Decisions in a second window; banking vs forum sensitivity (already
   in policy YAML). Out: transaction-time step-up, extra factor types, new
   services, GBM on the hot path, GitOps.

## Consequences

- Positive: the defense shows a person logging into an app, the PDP scoring
  it, and friction only when the login looks unlike that user — with a reason
  list. VPN users are not accused of flying. Country centroids avoid most
  CGNAT city noise.
- Negative: country-level travel will not catch same-country jumps; WebAuthn
  on a phone needs the phone to reach the IdP; a static VPN ASN list is a
  thesis stand-in for commercial IP intel.
- Follow-up: implement Demo-1 first (contracts/feature schema bump as needed).
  Update `tests/test_parity.py` when the travel flag lands. K8s-3 remains
  optional after the walkthrough exists.
