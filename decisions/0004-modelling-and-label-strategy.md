# ADR-0004: Modelling approach & label strategy

- Status: Accepted
- Date: 2026-08-08

## Context

EDA (see `findings/2026-08-08-phase1-eda.md`) showed the honest label
`is_account_takeover` is **extraordinarily rare: 141 positive rows in the entire
~31M-login dataset** (138 users). A chronological split leaves ~100 positives to
train on — far too few to trust a supervised classifier on that target. The more
populated `is_attack_ip` flag (8.7% of rows) exists but is leakage-sensitive and
represents a different target (attack IP ≠ account takeover).

## Decision

- **Primary scorer: Freeman et al. (2016) likelihood-ratio model.** It learns what
  *normal* per-user behavior looks like from the abundant legitimate data and flags
  deviations, so it needs almost no attack labels — a natural fit for label scarcity,
  and it gives native per-feature (explainable) contributions.
- `is_attack_ip` is **reserved for the leakage A/B comparison only** (Variant A with
  it vs Variant B without), not used as the training target.
- Supervised models (LogReg, RandomForest, LightGBM/XGBoost) are kept as **reference
  baselines** and for the SHAP explainability story, reported honestly given the
  label scarcity.

## Consequences

- The MVP can be credible with no ML at all (Freeman only).
- Evaluation must be **conditioned on history depth** (40% of users have a single
  login and cannot be protected by behavioral RBA yet — the "logins-to-protection"
  insight).
- Metrics: recall @ fixed low FPR, PR-AUC, challenge rate, per-history-depth
  breakdown — never plain accuracy.
