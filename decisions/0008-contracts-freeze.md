# ADR-0008: Freeze feature / model / PDP / event / policy contracts

- Status: Accepted
- Date: 2026-08-08

## Context

Phase 1 proved the RBA approach is feasible (Freeman + baselines, leakage A/B
negative, calibration β→5). Phase 3 will build `rba-decision-service` and later
async consumers. Before that, every repo needs a single, versioned agreement on
what crosses process boundaries — otherwise polyrepo drift recreates the same
class of failure that `rba-features` already prevents for train/serve skew
(ADR-0001).

Draft shapes existed only in exploratory notes (`material_sources/…`); nothing was
schema-frozen. Open questions included score scale (0–100 vs [0,1]), how Freeman’s
native `logrisk` maps onto a `predict_proba` policy input, which login signals are
in the v1 API, and the outbox event shape.

## Decision

Create **`rba-contracts` v0.1.0** (new polyrepo library) and freeze:

1. **Feature schema `FeatureVectorV1` (1.0.0)** — the ten names/types/order already
   implemented in `rba_features.features.FEATURE_NAMES`. Excluded on purpose:
   RTT, absolute geo / impossible travel, and `is_attack_ip` (ADR-0004).
2. **Model interface** — every scorer exposes `risk_score ∈ [0, 1]`
   (`ModelPrediction`) plus optional per-signal `contributions`. Freeman keeps
   `logrisk` as an optional native field; serving MUST declare
   `proba_mapping.method` on artifact metadata (default `logistic_logrisk`).
   Artifact sidecar JSON is required alongside any pickle/MLflow blob.
3. **`POST /risk/evaluate`** (OpenAPI 3.1) — PDP request/response. Required
   signals: `event_id`, `application_id`, `user_id`, `timestamp`, `ip_address`.
   Optional feature inputs match `rba-features` (`asn`, `country`, `device_type`,
   `os`, `browser`, `login_successful`). Actions:
   `ALLOW | REQUIRE_MFA | REAUTHENTICATE | BLOCK`. Levels:
   `LOW | MEDIUM | HIGH | CRITICAL`. Reasons are structured
   `{code, signal, contribution?, detail?}`.
4. **Event `rba.decision.made.v1`** (AsyncAPI 2.6) — outbox/bus payload; consumers
   idempotent on the same `event_id` supplied by the PEP.
5. **Policy config** — versioned YAML/JSON: `score_to_level` bands on the
   probability scale + `level_to_action` map + `fallback_action`, with optional
   per-`application_id` overrides (app sensitivity).

Executable Pydantic models ship in the same repo so FastAPI services import one
package; YAML/JSON Schema remain the language-neutral source of truth.

## Consequences

- Positive: Phase 3 can stub `/risk/evaluate` against a real contract; policy and
  model swaps do not change the PEP-facing API; `event_id` correlation is defined
  end-to-end; feature renames require an explicit schema version bump.
- Negative: Freeman online still needs per-value counts in `ProfileState` (noted in
  `freeman.py`) — that is a Phase 3 implementation follow-up, not a contract change.
  Generated non-Python clients are deferred until a second consumer language appears.
- Follow-up: publish `roku-pfi/rba-contracts` on GitHub; have decision-service and
  ml-training pin `rba-contracts==0.1.0`; keep the
  `FEATURE_NAMES` parity test green across both libraries.
