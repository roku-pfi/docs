# Freeman calibration — lifting the primary scorer's low-FPR recall

- Date: 2026-08-08
- Source: `rba-ml-training` — `python -m ml.calibrate`
  (scorer: `ml/models/freeman.py`; experiment: `ml/calibrate.py`).
- Regenerate: same command; report at `reports/freeman_calibration/metrics.md`.
- Related: ADR-0004 (Freeman primary — this refines its hyper-parameter),
  ADR-0007 (evaluation protocol), findings `2026-08-08-step5-baselines.md`.

## Goal

Step 5 left the primary Freeman scorer ranking well (ROC-AUC 0.76) but weak at the
strict operating point: **recall@1%FPR = 0.079 (3/38)**, because the equally-weighted
sum is dominated by the near-unique IP term and over-flags first-time-IP legit logins.
We tested three calibration levers on the identical chronological split — note that
plain probability calibration (Platt/isotonic) is monotone and cannot move recall@FPR,
so the lever has to change the *ranking*.

## What we tried, and what happened

| variant | labels? | ROC-AUC | recall@1%FPR | note |
|---|---|---|---|---|
| freeman (Step 5, beta=10) | no | 0.762 | 0.079 (3/38) | baseline |
| **freeman, beta=5** | no | **0.779** | **0.105 (4/38)** | **adopted (new default)** |
| freeman, beta=2 | no | 0.784 | 0.105 (4/38) | marginally higher AUC |
| freeman_noip (beta=10) | no | 0.711 | 0.105 (4/38) | AUC drops — not adopted |
| freeman_weighted | yes | 0.670 | 0.026 | **overfits — rejected** |
| freeman_weighted_noip | yes | 0.635 | 0.026 | overfits — rejected |

Smoothing (`beta`) sweep on the raw all-feature scorer:

| beta | ROC-AUC | recall@1%FPR |
|---|---|---|
| 2 | 0.784 | 0.105 |
| 5 | 0.779 | 0.105 |
| 10 (old default) | 0.762 | 0.079 |
| 20 | 0.756 | 0.079 |
| 50 | 0.752 | 0.079 |

## Findings

1. **Lowering the smoothing is the (modest, free) win.** Dropping `beta` 10 → 5 trusts
   each user's short history sooner, sharpening per-user distributions. It improves
   **both** ROC-AUC (0.762 → 0.779) and recall@1%FPR (3/38 → 4/38), needs **no labels**,
   and is theoretically clean. **Adopted as the new `FreemanScorer` default (beta=5).**
   beta=2 is marginally better on AUC but more aggressive; 5 is the robust pick.
2. **Dropping IP is a wash, not a fix.** It nudges the operating point (Δrecall +0.026,
   CI [0.000, 0.086]) but *hurts ranking* (ROC 0.762 → 0.711, CI clearly negative). The
   beta-tuned full-feature scorer dominates it (0.779 AUC at the same 0.105 recall), so
   we keep IP and just re-smooth.
3. **Supervised per-feature weighting overfits and is rejected.** A logistic layer on
   the 7 LLR contributions (learned from ~103 train positives) *worsens* test ROC
   (0.762 → 0.670) and recall (→ 0.026); it even learned counter-intuitive weights
   (`device_type` −0.69). With this few positives the weights don't generalise across
   the train→test time boundary — a concrete demonstration of the label-scarcity
   ceiling (dataset-sufficiency note), and why the *label-free* Freeman is the primary.
4. **The real ceiling is history depth, not weighting.** Every variant detects **0** of
   the 17 positives in the 0- and 1-2-login buckets; all detection lives in the
   ≥3-history bucket. Behavioural RBA cannot protect low-history users yet
   (logins-to-protection), so Freeman's recall@1%FPR is structurally capped around the
   ≥3-history positives (~0.4 max); calibration only shifts detections within that band.

## Decision / consequence

- **`FreemanScorer` default `beta` 10 → 5** (refines ADR-0004). `ml/train.py` picks this
  up automatically; the report pins the baseline to beta=10 for a clean before/after.
- **Reject** the supervised weighted variant and the IP-drop; keep the raw, label-free
  Freeman (now beta=5) as the primary explainable/ranking scorer.
- When the *operating point* is what matters and labels exist, the **Step 5 supervised
  LogReg (0.50 recall@1%FPR)** remains the better choice — the two are complementary
  (label-free ranking vs supervised detection), not competitors.

## Caveats

- 38 test positives → the recall deltas are literally ±1 detection and within noise;
  the beta change is adopted because it is principled and free, **not** because 4/38 is
  significantly better than 3/38.
- The per-feature contributions machinery (`FreemanScorer.contributions_frame`) added
  here is also what will power per-signal explanations online (Phase 3).
