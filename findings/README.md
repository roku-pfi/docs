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

Local copies of plots/metrics also land in gitignored
`rba-ml-training/reports/` — **this folder is the citable source**.

## How to add one

`docs/findings/YYYY-MM-DD-<slug>.md` with: source command, subset used,
results as tables, reproduction steps, data caveats. Then link it from
`plans/status.md` and a `devlog.md` entry.
