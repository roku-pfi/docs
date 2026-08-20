# Findings

Experiment and analysis write-ups with the **concrete numbers** the thesis will
cite. Tables preferred. Never invent figures — pull them from the command
output named in each file.

Reproduction lives in `rba-ml-training` (venv active). Dataset caveats:
Wiefling is a privacy-preserving synthesis of real SSO logins
([ADR-0003](../decisions/0003-dataset-selection-and-synthetic-data.md)).

## Index

| File | What | Command |
|---|---|---|
| [2026-08-08-dataset-sufficiency.md](2026-08-08-dataset-sufficiency.md) | Subset size vs full set; why 50k users is enough | subset builder |
| [2026-08-08-phase1-eda.md](2026-08-08-phase1-eda.md) | Imbalance, history depth, missingness, time span | `python -m ml.eda.explore` |
| [2026-08-08-step4-feature-validation.md](2026-08-08-step4-feature-validation.md) | Feature set + parity | `pytest` in `rba-features` |
| [2026-08-08-step5-baselines.md](2026-08-08-step5-baselines.md) | Freeman vs LogReg/RF/LightGBM (PR-AUC, recall@1% FPR) | `python -m ml.train --model all` |
| [2026-08-08-step6-leakage.md](2026-08-08-step6-leakage.md) | `is_attack_ip` Variant A vs B (no material leakage) | `python -m ml.leakage` |
| [2026-08-08-freeman-calibration.md](2026-08-08-freeman-calibration.md) | Dirichlet β 10→5; per-feature weights rejected | `python -m ml.calibrate` |
| [2026-08-16-k8s2-hpa-load.md](2026-08-16-k8s2-hpa-load.md) | PDP HPA 2→5 under load; p95/p99; Grafana | `rba-infra/scripts/load-hpa.sh` |
| [2026-08-19-supervised-escalation-and-failed-logins.md](2026-08-19-supervised-escalation-and-failed-logins.md) | Supervised escalate (0.395 vs Freeman 0.105); profile-poisoning fix and its cost; what LogReg learned | `python -m ml.train --model all` + `ml.export_logreg` |
| [2026-08-20-step5-rerun.md](2026-08-20-step5-rerun.md) | **Citable model numbers.** Full regenerated Step 5 table matching the deployed artifacts; why the two depth tables differ | `python -m ml.train --model all` |

> **Which numbers to cite:** `2026-08-20-step5-rerun.md`. It carries the full
> regenerated table and matches the operating point baked into the deployed
> `logreg-0.1.0.json`. The Step 5 (2026-08-08) results table and the calibration
> note predate ADR-0027's success-only familiarity and are superseded — logreg
> recall@1%FPR is **0.3947**, not 0.500; freeman **0.1053**, not 0.079.

Local copies of plots/metrics also land in gitignored
`rba-ml-training/reports/` — **this folder is the citable source**.

## How to add one

`docs/findings/YYYY-MM-DD-<slug>.md` with: source command, subset used,
results as tables, reproduction steps, data caveats. Then link it from
`plans/status.md` and a `devlog.md` entry.
