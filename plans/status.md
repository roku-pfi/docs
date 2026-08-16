# Project status — single source of truth

> The one place that says **where we are**. Update the checkboxes and the "Current
> focus" pointer whenever a step/phase completes. Narrative detail goes in
> [`../devlog.md`](../devlog.md) (newest on top); the phase rationale is in
> [`development_plan.md`](development_plan.md) §8 (**canonical roadmap** — product
> horizons in §1.1 / ADR-0012–0015); decisions in [`../decisions/`](../decisions/);
> numbers in [`../findings/`](../findings/).

Legend: `[x]` done · `[~]` in progress · `[ ]` not started.

## How we work now (RBA core, IdP shell)

**Thesis core = risk-based authentication.** Explainable login risk on a real
path: features → Freeman (or successor) → policy → `ALLOW` / `MFA` / `REAUTH` /
`BLOCK` + reasons. That is the research claim and the grade-critical work.
([ADR-0015](../decisions/0015-rba-is-the-thesis-core.md))

**Product shell = thesis-scale IdP** (Authentik/Auth0-shaped, not enterprise).
Demo apps are clients; people log in on the IdP; the IdP **asks the PDP** and
enforces the action. Admin shows users, apps, **decisions/reasons**, and policy
— not a user directory alone.
([ADR-0012](../decisions/0012-thin-idp-end-product.md),
[ADR-0014](../decisions/0014-thesis-scale-idp-platform.md))

The IdP without RBA is an empty login app. The PDP without the IdP is a risk
API nobody logs in through. We build the shell so the core is usable; we do
not let the shell replace the core.

| Like Auth0 / Authentik | Not like them | Thesis-specific |
|---|---|---|
| Users, apps, hosted login, session, MFA | Full OIDC/SAML, SCIM, LDAP, social login | Per-signal reasons, tunable policy, train/serve parity |
| Admin to operate the platform | Multi-tenant HA, billing, authz engines | Decision browser tied to the PDP |

**Working rule:** IdP stages are done. Each session now moves **one k8s/ops
stage** (below). Do not skip back into IdP niceties or federation. No identity
inside `decision-service`. Leftover Phase 4 items (DLQ) are not the current
path unless they unblock a k8s stage.

**Already the RBA core:** `rba-features`, `rba-contracts` (`/risk/evaluate` +
IdP login 0.2.0 + admin 0.3.0 + groups 0.4.0), `rba-decision-service`, async
profile/audit, `rba-infra` (compose + k3d/Helm + Prometheus/Grafana).
**Product shell:** `rba-idp` (IdP-1…7 done).

**Current focus:** K8s-3 — GitOps / reusable CI templates (Tilt optional).
Federation / SCIM / OIDC remain out.

## Phase roadmap (see `development_plan.md` §8)

- [x] **Phase 1 — Data & model feasibility**
- [x] **Phase 2 — Freeze contracts** (`rba-contracts` v0.1.0; ADR-0008)
- [x] **Phase 3 — Request path** (`rba-decision-service`; ADR-0009/0010). Local k8s
      via k3d/Helm (ADR-0020 / K8s-1).
- [~] **Phase 4 — Async services** ← thin slice + k8s Deployments done
      (ADR-0011). Remaining: DLQ / worker metrics.
- [~] **Phase 5 — Observability & load/scenario testing** ← Prometheus/Grafana
      + HPA load done (ADR-0021 / K8s-2). Remaining: event lag.
- [ ] **Phase 6 — ML lifecycle + generator**
- [~] **Phase 7 — Thin IdP platform + admin + k8s** (ADR-0012/0013/0014/0017/0019/0020;
      IdP-7 + K8s-1 done; K8s-2 done)
- [ ] **Phase 8 — Report & defense**
- [~] **Phase 0 — Infra foundations** — compose + k3d/Helm + Prometheus/Grafana
      done (ADR-0020/0021); Tilt / GitOps / CI templates still ahead

## Phase 1 — steps (done)

| # | Step | Status | Where | Result / finding |
|---|---|---|---|---|
| 1 | Environment + tooling (venv, deps, dataset acquired) | [x] | `rba-ml-training/.venv`, `data/README_personal.md` | ADR-0002 |
| 2 | Subset builder (stratified, sentinel-capped) | [x] | `ml/ingest/subset.py` | 202,284 rows; ADR-0005 |
| 3 | EDA (imbalance, history depth, missingness, span) | [x] | `ml/eda/explore.py` | findings `…phase1-eda.md` |
| 4 | Feature library + offline replay + parity test | [x] | `rba-features/` | findings `…step4-feature-validation.md` |
| 5 | Freeman scorer + reference baselines (RBA metrics) | [x] | `ml/train.py`, `ml/models/freeman.py`, `ml/metrics.py` | findings `…step5-baselines.md`; ADR-0007 |
| 6 | `is_attack_ip` leakage A/B comparison | [x] | `ml/leakage.py` | findings `…step6-leakage.md` (no material leakage) |
| + | Freeman calibration (smoothing tune; weighting rejected) | [x] | `ml/calibrate.py` | findings `…freeman-calibration.md` (β 10→5) |

**Regenerate the Phase-1 numbers:** from `rba-ml-training/` (venv active) —
`python -m ml.train --model all`, `python -m ml.leakage`, `python -m ml.calibrate`.

## Phase 2 — contracts (done)

Locked in `rba-contracts` v0.1.0 (ADR-0008); additive `login` snapshot in **v0.1.1**;
IdP login API in **v0.2.0** (IdP-1):

- [x] Feature schema / model I/O / `/risk/evaluate` / policy / `rba.decision.made.v1`
- [x] `LoginEventSnapshot` on `DecisionMadeEvent` (profile-service input)
- [x] IdP `POST /login` / `/mfa/verify` / `/session` / `/logout` (`idp.py`, `openapi/idp.yaml`)

## Phase 3 — request path (done aside from k8s)

- [x] FastAPI `POST /risk/evaluate` against `rba-contracts`
- [x] Redis profile + Freeman inline + policy + decision/outbox
- [x] Shared data plane via `rba-infra` (ADR-0010)
- [x] Exercised against real Redis/Postgres
- [x] Local k8s deploy (k3d + Helm, ADR-0020 / K8s-1)

## Phase 4 — async services (in progress)

- [x] RabbitMQ in `rba-infra`
- [x] `rba-event-publisher` drains outbox → `rba.events` / `rba.decision.made.v1`
- [x] `rba-profile-service` updates Redis + `rba_profile` history (idempotent)
- [x] `rba-audit-service` persists to `rba_audit` (idempotent)
- [x] E2E smoke: evaluate (`PROFILE_WRITE_MODE=none`) → publish → profile + audit
- [x] k8s Deployments (Helm chart in `rba-infra`, ADR-0020)
- [ ] GitHub remotes + CI for the three new repos
- [ ] DLQ / worker metrics

## Phase 7 — thin IdP (this is the product path)

Stages in order. All IdP-1…7 boxes are done; leftover Phase 7 is k8s.

- [x] **IdP-1** Contracts (`rba-contracts` 0.2.0)
- [x] **IdP-2** Identity store (`rba-idp` + users + seeded application, password verify)
- [x] **IdP-3** PDP enforce (call `/risk/evaluate`, map action → outcome)
- [x] **IdP-4** Session + mock MFA
- [x] **IdP-5** Hosted login UI (Auth0/Authentik-style login page)
- [x] **IdP-6** Admin console (users, applications, decisions, policy)
- [x] **IdP-7** Stretch: groups / app-scoped permissions (still no OIDC/SAML/SCIM)

## k8s stages (Phase 0 remainder / Phase 5 / Phase 7 leftover)

- [x] **K8s-1** Local k3d cluster + Helm data plane + service Deployments
      ([ADR-0020](../decisions/0020-local-k8s-k3d-helm.md)). IdP Ingress on
      `:8080`; PDP 2 replicas + HPA; compose still valid without a cluster.
- [x] **K8s-2** Prometheus / Grafana + load test that moves the HPA (Phase 5)
      ([ADR-0021](../decisions/0021-prometheus-grafana-in-chart.md);
      finding [`2026-08-16-k8s2-hpa-load.md`](../findings/2026-08-16-k8s2-hpa-load.md)).
- [ ] **K8s-3** GitOps / reusable CI templates (Tilt optional)
