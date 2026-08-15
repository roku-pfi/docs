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

**Working rule:** each session moves **one IdP stage**
([ADR-0013](../decisions/0013-idp-staged-start.md)), but **IdP-3 (PDP enforce)
is not skippable** — that is where RBA re-enters the login. No throwaway PEP
scripts, no identity inside `decision-service`. Leftover Phase 4/5/6/0 items
are not the current path unless they unblock an IdP stage.

**Already the RBA core:** `rba-features`, `rba-contracts` (`/risk/evaluate` +
IdP login 0.2.0), `rba-decision-service`, async profile/audit, `rba-infra`.
**Product shell started:** `rba-idp` (IdP-2: users + seeded app + password verify).

**Current focus:** **IdP-3** — `rba-idp` calls `POST /risk/evaluate` and maps
action → login outcome + reasons. No session/OTP yet (that is IdP-4).

## Phase roadmap (see `development_plan.md` §8)

- [x] **Phase 1 — Data & model feasibility**
- [x] **Phase 2 — Freeze contracts** (`rba-contracts` v0.1.0; ADR-0008)
- [x] **Phase 3 — Request path** (`rba-decision-service`; ADR-0009/0010). Local k8s
      still waits on fuller Phase 0 Helm.
- [~] **Phase 4 — Async services** ← thin slice done (publisher + profile + audit;
      ADR-0011). Remaining: DLQ/metrics/k8s Deployments.
- [ ] **Phase 5 — Observability & load/scenario testing**
- [ ] **Phase 6 — ML lifecycle + generator**
- [~] **Phase 7 — Thin IdP platform + admin + k8s** (ADR-0012/0013/0014; IdP-2 done)
- [ ] **Phase 8 — Report & defense**
- [~] **Phase 0 — Infra foundations** — compose (Redis/Postgres/RabbitMQ) done;
      k3d/Tilt/Helm/CI still ahead

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
- [ ] Local k8s deploy (Phase 0 Helm)

## Phase 4 — async services (in progress)

- [x] RabbitMQ in `rba-infra`
- [x] `rba-event-publisher` drains outbox → `rba.events` / `rba.decision.made.v1`
- [x] `rba-profile-service` updates Redis + `rba_profile` history (idempotent)
- [x] `rba-audit-service` persists to `rba_audit` (idempotent)
- [x] E2E smoke: evaluate (`PROFILE_WRITE_MODE=none`) → publish → profile + audit
- [ ] GitHub remotes + CI for the three new repos
- [ ] DLQ / metrics / k8s Deployments

## Phase 7 — thin IdP (this is the product path)

Stages in order. Next unchecked box is the only IdP work to pick up.

- [x] **IdP-1** Contracts (`rba-contracts` 0.2.0)
- [x] **IdP-2** Identity store (`rba-idp` + users + seeded application, password verify)
- [ ] **IdP-3** PDP enforce (call `/risk/evaluate`, map action → outcome) ← next
- [ ] **IdP-4** Session + mock MFA
- [ ] **IdP-5** Hosted login UI (Auth0/Authentik-style login page)
- [ ] **IdP-6** Admin console (users, apps, decisions, policy)
- [ ] **IdP-7** Stretch: groups / app-scoped permissions (still no OIDC/SAML/SCIM)
