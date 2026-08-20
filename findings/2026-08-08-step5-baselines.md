# Step 5 — Freeman scorer + reference baselines

> **The results table below is superseded.** These are the numbers from the
> 2026-08-08 training run. The artifacts actually deployed on the request path
> were retrained after ADR-0027 (success-only familiarity) and score differently
> — logreg recall@1%FPR **0.3947**, not 0.500; freeman **0.1053**, not 0.079.
> **Cite [`2026-08-20-step5-rerun.md`](2026-08-20-step5-rerun.md).** The
> evaluation *design* recorded here — label window, chronological split,
> ADR-0007 — is unchanged and still current.

- Date: 2026-08-08
- Source: `rba-ml-training` — `python -m ml.train --model all`
  (Freeman: `ml/models/freeman.py`; baselines + eval: `ml/train.py`, `ml/metrics.py`).
- Regenerate: same command; report written to `reports/step5/metrics.md`.

## Key data finding (drove the evaluation design)

Account-takeover labels only cover **2020-02-04 → 2020-11-23**; there are **zero
positives after Nov 2020**, even though `is_attack_ip` continues all year. A plain
calendar 70/15/15 split therefore leaves the test tail with **0 positives** (metrics
undefined). We instead **restrict to the label-covered window and split chronologically
within it** (train 70% / test 30%) — see ADR-0007. Excluded 49,191 later unlabeled rows.

- Test set: **45,928 logins, 38 positives** (train: ~103 positives).

## Results (test set, honest chronological split)

| Model | ROC-AUC | PR-AUC | Recall @ FPR≈1% | Challenge rate | Size | Latency |
|---|---|---|---|---|---|---|
| **logreg** (baseline) | **0.880** | 0.034 | **0.500** (19/38) | 0.9% | 1.7 KB | 0.1 µs |
| freeman (PRIMARY) | 0.762 | 0.004 | 0.079 (3/38) | 0.9% | 798 KB | 5.3 µs |
| rf | 0.693 | 0.028 | 0.026 | 0.0% | 7.9 MB | 1.4 µs |
| lgbm | 0.554 | 0.001 | 0.000 | 0.0% | 47 KB | 0.1 µs |

### Recall by history depth (at the 1%-FPR threshold)

| bucket | n | positives | logreg recall | freeman recall |
|---|---|---|---|---|
| 0 (first login) | 8,750 | 8 | 0.00 | 0.00 |
| 1-2 | 10,343 | 9 | 0.89 | 0.00 |
| 3-9 | 14,615 | 15 | 0.60 | 0.20 |
| 10+ | 12,220 | 6 | 0.33 | 0.00 |

## Takeaways

- **The RBA approach works on real data.** Ranking is well above chance (ROC-AUC
  0.76–0.88). A simple, explainable **LogisticRegression on the `*_seen_before` feature
  vectors reaches 50% recall at a 1% false-positive rate** — a genuinely usable operating
  point given only 38 test positives.
- **Freeman ranks reasonably (AUC 0.76) with no labels**, which is its value under label
  scarcity, but its raw IP-dominated score over-flags first-time-IP legit logins, so it
  underperforms at the strict low-FPR point. Next improvement: per-feature weighting /
  calibration and down-weighting near-unique IP. (It remains the label-free primary.)
- **Trees (RF/LGBM) underperform LogReg here** — the mostly-binary seen-before features
  are near-linear in log-odds, and extreme rarity makes the trees overfit; LogReg is the
  strong simple baseline. (LGBM defaults are clearly poor here; not worth tuning for the
  MVP.)
- **History depth is decisive** (confirms EDA): detection lives in users with ≥1 prior
  login; the 8 first-login positives are undetectable by behavioural RBA — the
  "logins-to-protection" story.
- **Numbers are noisy** (38 positives) — report them as feasibility evidence with CIs
  implied, not as tuned performance.

## Caveats

- Small positive count → wide uncertainty; do not over-interpret single-model deltas.
- Freeman per-user counts are maintained offline here; serving needs them in
  `rba_features.ProfileState` (Phase 3 parity follow-up).
- `is_attack_ip` was NOT used as a feature here; the leakage A/B is Step 6.
