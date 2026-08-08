# Development log

Reverse-chronological. Newest first. Each entry: what we did, why, and findings.

---

## 2026-08-08 — Step 6: `is_attack_ip` leakage A/B (negative result)

Ran the mandatory leakage experiment (`ml/leakage.py`): each supervised baseline trained
twice on the same chronological split — Variant B (behavioural `rba_features` vectors
only) vs Variant A (B + `is_attack_ip`) — with paired bootstrap CIs (findings
2026-08-08-step6-leakage.md).

**Result: no material leakage.** In this subset the flag is weak/noisy, not an oracle:
`is_attack_ip`=1 on only **47% of takeovers but 8.45% of legit** logins, and the flag
alone scores ROC-AUC 0.70 with **recall@1%FPR = 0**. Our best honest model (logreg)
gains nothing from it — ΔROC-AUC +0.003 (CI [−0.02, 0.03]), Δrecall@1%FPR −0.053. So the
Step 5 headline (logreg 0.88 ROC-AUC, 50% recall @ 1% FPR) is genuinely behavioural, not
inflated by the attack-derived label. Decision (executing ADR-0004): keep `is_attack_ip`
out of the trained model — now evidence-backed, not just precaution.

Refactored `ml/train.py` to share a `build_models(ytr)` factory so Variant B reproduces
Step 5 hyper-parameters exactly.

**Next:** Phase 1 wrap-up — optionally Freeman calibration to lift low-FPR recall, then
lock the feature/model/scoring contracts (Phase 2) before the request path.

---

## 2026-08-08 — Dataset sufficiency note (for the report)

Captured the reasoning on whether the Wiefling dataset is "good enough"
(findings/2026-08-08-dataset-sufficiency.md): it is valid (standard benchmark, faithful
structure) but positive labels are structurally scarce — **141 takeovers is the global
ceiling**, not a sampling artifact, and the scarcity is realistic. Sufficient for
proving the mechanism + driving the system engineering; thin only for tuned detection
metrics, which we handle via the label-free Freeman primary, `is_attack_ip` as a
populous auxiliary signal, a later synthetic scenario generator (complement), and
bootstrap CIs on metrics.

---

## 2026-08-08 — Step 5: Freeman scorer + baselines (feasibility numbers)

Implemented the Freeman likelihood-ratio scorer (`ml/models/freeman.py`), the baseline
trainer/evaluator (`ml/train.py`), RBA metrics (`ml/metrics.py`), and the shared
feature-matrix builder (`ml/featurize.py`).

Key data finding: **takeover labels only cover Feb–Nov 2020** (0 positives afterwards),
so a calendar split gave a test set with 0 positives. Fixed by restricting to the
label-covered window and splitting chronologically within it (ADR-0007). Test set:
45,928 logins / 38 positives.

Results (findings/2026-08-08-step5-baselines.md): the RBA approach works — ROC-AUC
0.76–0.88. Simple **LogReg on `*_seen_before` features hits 50% recall @ 1% FPR**;
Freeman ranks at AUC 0.76 label-free but over-flags new-IP legit logins at the strict
operating point (calibration is the next improvement). Trees underperform LogReg.
Detection concentrates in users with ≥1 prior login (history-depth story confirmed).

**Next:** Step 6 — `is_attack_ip` leakage A/B comparison + finalise metrics/latency.

---

## 2026-08-08 — AI-assisted development tooling (rules, skill, MCP)

Set up persistent tooling so guardrails and documentation habits survive across sessions
(ADR-0006). Added five Cursor rules in `develop/.cursor/rules/` (project overview,
feature-parity contract, ML data/evaluation discipline, documentation loop, git/polyrepo
conventions), mirrored to `docs/rules/`. Added the `/log-progress` skill to automate the
devlog+ADR+findings loop. Wired the GitHub MCP in `develop/.cursor/mcp.json` (auth via a
`GITHUB_PAT` env var). Other MCPs (Postgres/k8s/Grafana/MLflow) deferred to Phase 3+.

**Next:** Step 5 — Freeman likelihood-ratio scorer + reference baselines on a
chronological 70/15/15 split.

---

## 2026-08-08 — Step 4: feature library implemented & validated

Implemented the Phase 1 feature set in `rba-features`: `profile.ProfileState`
(per-user accumulator), `features.compute_features`/`update_profile` (10 features:
`user_login_count`, `{ip,asn,country,device_type,os,browser,hour}_seen_before`,
`seconds_since_last_login`, `failed_logins_last_24h`), and the `replay_user` driver.
Missing values ("-"/NaN) never count as "seen". 5 tests pass, incl. offline↔online
parity. RTT and absolute geo distance excluded (EDA: RTT ~94% missing, geo synthetic).

Validated on the real subset (findings/2026-08-08-step4-feature-validation.md): the
`*_seen_before` features strongly separate takeovers from normal logins
(ip_seen_before 0.08 vs 0.45; country 0.18 vs 0.75) — confirms the RBA hypothesis
and the Freeman-primary direction.

**Next:** Step 5 — Freeman likelihood-ratio scorer + reference baselines on a
chronological split.

---

## 2026-08-08 — Phase 1 kickoff: environment, dataset, subset, EDA

**Environment decision.** Consolidated all development onto the M2 Pro MacBook
(dropped the plan to train on a separate Windows/RTX 3070 PC): GBDT on tabular data
is CPU-bound and barely benefits from GPU, and one machine removes the two-machine
git-sync overhead. See ADR-0002 note.

**Tooling set up (local).** Installed `libomp` + Python 3.12.13 via Homebrew; created
`rba-ml-training/.venv` and installed deps. Verified imports: pandas 3.0.5, numpy
2.4.6, scikit-learn 1.9.0, lightgbm 4.7.0, xgboost 3.4.0, shap 0.52.0, rba_features
0.1.0 (editable).

**Dataset.** Downloaded the Wiefling RBA dataset (Zenodo 6782156, `rba-dataset.csv`,
8.4 GB / 16 cols). Header matches `rba_features.schema` exactly. Documented its
synthesized-but-real-derived nature (ADR-0003; `data/README_personal.md`).

**Step 2 — subset builder** (`ml/ingest/subset.py`). Memory-safe two-pass streaming;
samples whole users, preserves all takeover users. First run exposed a sentinel user
with 14M logins (98.6% of rows) → added `--max-user-logins` cap to drop non-human
accounts (ADR-0005). Result: 50k users → **202,284 rows**, mean ~4 logins/user, all
141 takeover rows preserved.

**Step 3 — EDA** (`ml/eda/explore.py`). Full results in
`findings/2026-08-08-phase1-eda.md`. Headlines: takeover label is 0.07% (141 rows
total); `is_attack_ip` 8.7%; 40% of users have a single login; RTT 93.8% missing;
region/city ~42% missing; IP/user-agent near-unique; 391-day clean time span.

**Modelling decision.** Given 141 positives, chose the Freeman likelihood-ratio model
as the primary scorer; `is_attack_ip` reserved for the leakage A/B comparison
(ADR-0004).

**Committed & pushed:** `rba-ml-training@45a5e56` (subset + EDA) to
`github.com/roku-pfi/rba-ml-training`.

**Next:** Step 4 — implement the feature set in `rba-features` + offline replay
driver + parity test.

---

## 2026-08-05 — Plan & architecture direction

**Development plan** created at `plans/development_plan.md`. Initial proposal was a
modular monolith; revised after feedback to a **polyrepo + microservices + Kubernetes**
target with a lean login path (ADR-0001).

**Repos scaffolded:** `rba-features` (shared feature library) and `rba-ml-training`
(offline pipeline), each its own git repo, later pushed to the `roku-pfi` GitHub org.

**Data documentation:** added the dataset acquisition guide and clarified that the
Wiefling dataset is a privacy-preserving synthesis of real data, not rule-invented
synthetic data (ADR-0003).
