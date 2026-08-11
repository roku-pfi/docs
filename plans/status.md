# Project status — single source of truth

> The one place that says **where we are**. Update the checkboxes and the "Current
> focus" pointer whenever a step/phase completes. Narrative detail goes in
> [`../devlog.md`](../devlog.md) (newest on top); the phase rationale is in
> [`development_plan.md`](development_plan.md) §8; decisions in
> [`../decisions/`](../decisions/); numbers in [`../findings/`](../findings/).

**Current focus:** Phase 3 request path + light Phase 0. Thin slice PDP is up;
shared Redis/Postgres now live in **`rba-infra`** (ADR-0010). **Remaining:**
exercise decision-service against that stack; harden fallbacks; full k3d/Tilt later.
**Next after Phase 3:** Phase 4 async (outbox → RabbitMQ, profile-service, audit).

Legend: `[x]` done · `[~]` in progress · `[ ]` not started.

## Phase roadmap (see `development_plan.md` §8)

- [x] **Phase 1 — Data & model feasibility**
- [x] **Phase 2 — Freeze contracts** (`rba-contracts` v0.1.0; ADR-0008)
- [~] **Phase 3 — Request path** ← in progress (`rba-decision-service` thin slice;
      ADR-0009). Shared compose moved to `rba-infra` (ADR-0010).
- [ ] **Phase 4 — Async services** (event-publisher/outbox → RabbitMQ; profile-service,
      audit-service consumers)
- [ ] **Phase 5 — Observability & load/scenario testing** (Prometheus/Grafana; load
      tests that trigger the HPA — the scalability demo)
- [ ] **Phase 6 — ML lifecycle + generator** (dataset-builder, ml-training Job, MLflow;
      split model-inference to a sidecar; synthetic scenario generator)
- [ ] **Phase 7 — k8s hardening + frontend + admin** (HPA, probes, limits, GitOps;
      admin-api + demo frontend)
- [ ] **Phase 8 — Report & defense** (ongoing; feed the thesis continuously)
- [~] **Phase 0 — Infra foundations** — light bootstrap done (`rba-infra` compose);
      k3d/Tilt/Helm/CI/Prometheus still ahead

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

Locked in `rba-contracts` v0.1.0 (ADR-0008):

- [x] Feature schema (`FeatureVectorV1` / `FEATURE_NAMES` order, schema 1.0.0).
- [x] Model interface (`ModelPrediction` / `predict_proba` in [0,1] + artifact metadata).
- [x] `/risk/evaluate` request/response (`openapi/risk-evaluate.yaml`).
- [x] Event contract (`asyncapi/decision-events.yaml` — `rba.decision.made.v1` / `event_id`).
- [x] Config format: score→level and level→action (`schemas/policy-config.schema.json`).

## Phase 3 — request path (in progress)

`rba-decision-service` v0.1.0 (ADR-0009):

- [x] FastAPI `POST /risk/evaluate` against `rba-contracts`.
- [x] Profile read (Redis or in-memory) via serialised `ProfileState` (+ Freeman counts).
- [x] Features via `rba-features.compute_features`.
- [x] Freeman inline from JSON serving artifact (`logistic_logrisk`, β=5).
- [x] Policy + structured reasons; fallback → `fallback_action`.
- [x] Decision + outbox row in one DB transaction; idempotent on `event_id`.
- [x] Parity / unit tests (6 passed; optional ml-training cross-check skipped without pandas).
- [x] Shared Redis/Postgres via `rba-infra` compose (ADR-0010); service-local compose removed.
- [ ] Exercise evaluate against real Redis/Postgres day-to-day.
- [ ] Local k8s deploy (waits on fuller Phase 0 / Helm).
