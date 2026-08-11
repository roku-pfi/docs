# ADR-0009: Online ProfileState counts + Freeman JSON serving artifact

- Status: Accepted
- Date: 2026-08-11

## Context

ADR-0008 froze the PDP contracts and noted that online Freeman scoring needs
**per-value user counts**, while `rba_features.ProfileState` only stored
seen-sets. Phase 3 (`rba-decision-service`) cannot score with train/serve parity
without closing that gap. Pickling `ml.models.freeman.FreemanScorer` into the
hot path would also couple the PDP to the training package and pandas.

## Decision

1. **Extend `ProfileState`** with `freeman_counts` / `freeman_totals`, updated
   inside `update_profile` (same compute-then-update contract). Missing
   categoricals are skipped (never inflate counts), consistent with seen-set
   policy. Add `profile_to_dict` / `profile_from_dict` for Redis JSON.
2. **Serve Freeman from a JSON artifact** (`global_counts`, `alpha`, `beta`,
   vocab) produced by `python -m ml.export_freeman`. Decision-service loads it
   into `FreemanOnlineScorer` — no pickle, no `rba-ml-training` import on the
   request path. Default serving `beta=5.0` (calibration finding).
3. **`risk_score = logistic(logrisk)`** as declared in contracts
   (`proba_mapping.method = logistic_logrisk`).
4. **Phase 3 profile writes:** `PROFILE_WRITE_MODE=sync` updates Redis after each
   decision so demos/parity work before `profile-service` exists. Flip to `none`
   in Phase 4 when the async consumer owns materialisation.

## Consequences

- Positive: online LLRs use the same math as offline `score_event`; Redis payload
  is explicit; PDP stays a thin FastAPI service.
- Negative: Redis keys grow with distinct IPs/ASNs per user (acceptable for the
  thesis scale). Offline `contributions_frame` still stringifies `"-"` via
  pandas; online count updates skip missing — documented divergence only for
  missing-valued history rows.
- Follow-up: Phase 4 outbox → profile-service; optional sidecar for the model
  (Phase 6).
