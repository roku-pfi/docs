# ADR-0007: Evaluation protocol (label-covered window + chronological split)

- Status: Accepted
- Date: 2026-08-08

## Context

Step 5 revealed that account-takeover labels only cover 2020-02-04 → 2020-11-23; there
are zero positives in the last ~3 months, while `is_attack_ip` spans the full year. A
naive calendar 70/15/15 chronological split puts all 141 positives in train/val and
leaves the test set with **0 positives**, making every detection metric undefined. We
must still evaluate honestly (no temporal leakage) despite the scarce, time-bounded label.

## Decision

- **Restrict evaluation to the label-covered window** (up to the last positive
  timestamp); exclude later unlabeled rows from the modelling split.
- **Split chronologically within that window** (default train 70% / test 30%) so both
  sides contain positives and no future information leaks into training.
- **Metrics**: ROC-AUC and PR-AUC (ranking); recall at a fixed low FPR (≈1%) with the
  resulting challenge rate (operating point); everything broken down **by history depth**.
  Operating point is computed via `roc_curve` (correct tie handling), never plain accuracy.
- **Freeman** fits its global/population model on the train window only and scores
  past-only per user; it never uses the label, so evaluating it on the label needs no
  additional split.

## Consequences

- Positive: honest, leakage-free metrics with positives on both sides; the label-window
  limitation is documented rather than hidden.
- Negative: only ~38 test positives → wide uncertainty; report as feasibility evidence,
  not tuned performance. ~49k later rows are unusable for supervised evaluation.
- Follow-up: if more positives are needed, consider stratified time-aware resampling or
  augmenting with the synthetic scenario generator (later phase) — kept separate from
  this real-data backbone.
