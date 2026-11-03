# Backtest: the adopted model on takeovers no choice was made on (C1)

- Date: 2026-11-03
- Source: `rba-ml-training` — `python -m ml.backtest` (→ `reports/backtest.md`).
- Related: [hierarchical Freeman](2026-10-23-hierarchical-freeman.md),
  [Step 5](2026-08-08-step5-baselines.md), [policy comparison](2026-10-22-policy-comparison.md).

## Why

Every version choice (freeman-0.2 → 0.3 → 0.4, dropping the hour, the hybrid
over the 22/38 variant) was read off the Step 5 test split: the last 30% of the
label window, 38 takeovers. The reviewer's C1: the 20/38 may be optimistic.

The other **103 takeovers** lie before that split. None was used for a choice:
Freeman learns without labels, and no metric was ever computed on them. So the
adopted configuration is frozen and replayed earlier in time: global tables
from a training window, every user scored past-only, measured on the next
window. 95% intervals resample users (138 distinct victims for 141 takeovers).

## A. Same metric as the reported 20/38 (recall at 1% FPR of the window)

| training → evaluated (quantiles of time) | takeovers | adopted 0.4.0 hybrid | AUC | rejected "22/38" variant | flat 0.3.0 |
|---|---|---|---|---|---|
| 30% → 50% | 30 | **12/30 (0.40; 0.23–0.59)** | 0.808 | 17/30 | 4/30 |
| 40% → 70% | 48 | **21/48 (0.44; 0.30–0.58)** | 0.811 | 25/48 | 7/48 |
| 50% → 70% | 29 | **12/29 (0.41; 0.23–0.61)** | 0.801 | 16/29 | 5/29 |
| 70% → 100% (Step 5 test, used for choices) | 38 | 20/38 (0.53; 0.37–0.68) | 0.889 | 22/38 | 5/38 |

The reference row reproduces the published numbers exactly, so the backtest
runs the same pipeline.

## B. Does a 1% threshold carry over in time?

Threshold at 1% FPR on a calibration slice the tables never saw, applied
unchanged to the next window: **18–25% of legitimate logins** cross it (recall
0.61–0.67).

## What it means

- **The 20/38 is optimistic, but not invented.** On untouched takeovers the
  adopted model catches 40–44% at 1% FPR (AUC ≈ 0.81), against 53% (0.889) on
  the test split. Every interval overlaps the published one.
- **Much of the gap is the period, not the selection.** The earlier flat model
  loses the same AUC (0.807 → 0.70–0.74) between the test split and the earlier
  windows: later in time users have more history, which helps every variant.
- **The ranking holds.** Hierarchical ≫ flat in every window (≈3×). The
  rejected variant stays ahead (+4 to +5 takeovers); it was rejected for a
  deployment reason that the pseudonymized dataset cannot show (popularity of a
  browser name has no real meaning there), so this is expected, and the cost of
  that choice is now quantified.
- **Thresholds are not portable.** A score cut-off tuned on one period flags a
  fifth of the next period's logins. That is the same order as the policy
  comparison's 21.4% challenge rate, and the reason the deployment learns its
  own population (ADR-0042) and starts tenants in monitor mode (ADR-0034).
  The supervised threshold 0.869 was not backtested (it needs labels in the
  training window); it stays declared as measured on the test split.
