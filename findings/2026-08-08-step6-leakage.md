# Step 6 — `is_attack_ip` leakage A/B experiment

- Date: 2026-08-08
- Source: `rba-ml-training` — `python -m ml.leakage`
  (implementation: `ml/leakage.py`; shared model factory: `ml/train.build_models`).
- Regenerate: same command; report written to `reports/step6/leakage.md`.
- Related: ADR-0004 (label strategy — `is_attack_ip` reserved for this test),
  ADR-0007 (evaluation protocol), plan §6/§7.1, findings `2026-08-08-step5-baselines.md`.

## What we tested (and why it is mandatory)

`is_attack_ip` is derived from the attack itself, so a supervised model handed that
column can score well while learning nothing about user behaviour. The plan mandates
training every baseline twice on the **same** chronological split — **Variant B**
(behavioural/context `rba_features` vectors only, the honest number) and **Variant A**
(Variant B **+ `is_attack_ip`**) — and reporting B if A ≫ B. Freeman is label-free and
never consumes the flag, so it is inherently Variant B and excluded from the A/B.

Test set: **45,928 logins / 38 positives** (identical split to Step 5). Every metric —
and the paired A−B delta — carries a 95% percentile bootstrap CI (2,000 resamples),
because 38 positives make single deltas noisy.

## Headline: there is **no material leakage** in this subset

The flag is a **weak, noisy signal here, not a clean oracle**:

- **`is_attack_ip`=1 on only 47% of takeovers, but on 8.45% of legit logins.** It
  misses over half the attacks and fires on ~1 in 12 normal logins.
- **`is_attack_ip` alone: ROC-AUC 0.70, recall@1%FPR = 0.** Its own FPR (8.45%) is far
  above any usable operating point, so as a standalone rule it detects nothing at 1%
  FPR while over-challenging legit users.

### Variant B (honest) vs Variant A (+is_attack_ip)

| Model | ROC-AUC B | ROC-AUC A | recall@1%FPR B | recall@1%FPR A |
|---|---|---|---|---|
| **logreg** (best) | **0.880** | 0.883 | **0.500** | 0.447 |
| rf | 0.693 | 0.758 | 0.026 | 0.000 |
| lgbm | 0.554 | 0.489 | 0.000 | 0.000 |

### A − B deltas (paired bootstrap, 95% CI)

| Model | ΔROC-AUC (A−B) | ΔROC-AUC CI | Δrecall@1%FPR (A−B) | Δrecall CI |
|---|---|---|---|---|
| logreg | +0.003 | [−0.020, 0.030] | −0.053 | [−0.138, 0.000] |
| rf | +0.065 | [0.003, 0.126] | −0.026 | [−0.093, 0.000] |
| lgbm | −0.065 | [−0.151, 0.025] | 0.000 | [0.000, 0.000] |

## Takeaways

- **Our best honest model (logreg) gains nothing from the flag.** ΔROC-AUC CI straddles
  0 and Δrecall@1%FPR is *negative* — the `*_seen_before` behavioural features already
  capture (and exceed) whatever weak signal `is_attack_ip` carries. So the Step 5
  headline (**logreg, 0.88 ROC-AUC, 50% recall @ 1% FPR**) is **not** inflated by
  leakage; it is a genuine behavioural result.
- **The only positive delta (rf ROC-AUC +0.065, CI barely clears 0) does not survive at
  the operating point** (Δrecall −0.026) and rf is a weak model to begin with (0.69).
  Nothing indicating a model that "wins by memorising bad IPs."
- **Why so little leakage?** In this privacy-preserving synthesis the attack IPs are not
  disjoint from legit traffic (8.45% legit base rate) and only half of takeovers use a
  flagged IP. The flag behaves like coarse, noisy IP reputation — useful context at
  best, not a label shortcut. This matches the dataset-sufficiency note: the honest
  signal lives in per-user novelty, not in the attack flag.

## Decision / consequence

- **Keep `is_attack_ip` out of the trained model** (Variant B is the thesis number),
  now backed by evidence rather than caution. It may later serve as a low-weight
  *context* feature or a deterministic rule input, but never as a training target and
  never as the primary signal.
- The leakage experiment is **complete and negative**, which is the good outcome: it
  lets the thesis state the reported detection numbers are behavioural, not artefacts of
  an attack-derived label.

## Caveats

- 38 test positives → wide CIs; read deltas as "no material effect," not "exactly zero."
- Finding is specific to the Wiefling synthesis; on a dataset with a cleaner attack-IP
  oracle the leakage could be large, so the A/B stays part of the pipeline.
