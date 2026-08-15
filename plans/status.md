# Project status — single source of truth

> The one place that says **where we are**. Update the checkboxes and the "Current
> focus" pointer whenever a step/phase completes. Narrative detail goes in
> [`../devlog.md`](../devlog.md) (newest on top); the phase rationale is in
> [`development_plan.md`](development_plan.md) §8 (**canonical roadmap** — product
> horizons in §1.1 / ADR-0012); decisions in [`../decisions/`](../decisions/);
> numbers in [`../findings/`](../findings/).

**Product target:** Horizon A (near-term) = PDP demo; Horizon B (late Oct) = thin
IdP + admin wrapping the same PDP ([ADR-0012](../decisions/0012-thin-idp-end-product.md)).

**Current focus:** Phase 4 async thin slice is up (ADR-0011). Next for the ~10-day
demo: polish Horizon A (compose + evaluate + optional async), **without** starting
`rba-idp`. After that demo: Phase 5/0 as needed, then Phase 7 thin IdP.

Legend: `[x]` done · `[~]` in progress · `[ ]` not started.

## Phase roadmap (see `development_plan.md` §8)

- [x] **Phase 1 — Data & model feasibility**
- [x] **Phase 2 — Freeze contracts** (`rba-contracts` v0.1.0; ADR-0008)
- [x] **Phase 3 — Request path** (`rba-decision-service`; ADR-0009/0010). Local k8s
      still waits on fuller Phase 0 Helm.
- [~] **Phase 4 — Async services** ← thin slice done (publisher + profile + audit;
      ADR-0011). Remaining: DLQ/metrics/k8s Deployments.
- [ ] **Phase 5 — Observability & load/scenario testing**
- [ ] **Phase 6 — ML lifecycle + generator**
- [ ] **Phase 7 — Thin IdP + admin + k8s hardening** (ADR-0012; groups/permissions
      stretch)
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

Locked in `rba-contracts` v0.1.0 (ADR-0008); additive `login` snapshot in **v0.1.1**:

- [x] Feature schema / model I/O / `/risk/evaluate` / policy / `rba.decision.made.v1`
- [x] `LoginEventSnapshot` on `DecisionMadeEvent` (profile-service input)

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

## Phase 7 preview — thin IdP (not started; ADR-0012)

- [ ] `rba-idp` + `rba_idp` DB in infra
- [ ] Admin: users, decisions, policy
- [ ] Stretch: groups / app-scoped permissions
- [ ] IdP/admin contracts in `rba-contracts`
