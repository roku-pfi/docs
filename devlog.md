# Development log

Reverse-chronological. Newest first. Each entry: what we did, why, and findings.

---

## 2026-08-15 — Polyrepo README pass

Read every checkout in `develop/` and expanded documentation so each directory
has an accurate README (layout, APIs, env, status). No behaviour change.

- Workspace map: `develop/README.md` (polyrepo is not a single git repo).
- Service READMEs: features, contracts, decision-service, IdP, event-publisher,
  profile-service, audit-service, infra, ml-training (`data/README.md` added;
  the old `data/README.md` link was broken).
- Docs indexes: `docs/README.md`, `plans/`, `findings/`, `rules/`;
  `material_sources/README.md` points at the proposal PDF vs the living plan.

**Next:** IdP-5 — hosted login UI.

---

## 2026-08-15 — IdP-4 session + mock MFA (`rba-idp`)

Login now finishes the PEP loop: `ALLOW` mints an opaque bearer session;
`REQUIRE_MFA` / `REAUTHENTICATE` mint a `challenge_id`; `BLOCK` stays rejected
with neither. Completing the mock OTP (`000000`) issues the same session
without calling the PDP again.

- `POST /mfa/verify`, `GET /session` (`Authorization: Bearer`), `POST /logout`
  (204, idempotent).
- Session tokens stored hashed; challenges expire (5m) and cannot be reused.
- Wrong OTP → `INVALID_CREDENTIALS` (challenge stays open). Unknown/expired
  challenge → HTTP 400. Missing/expired session → HTTP 401.
- No hosted HTML — that is IdP-5.

**Next:** IdP-5 — hosted login UI.

---

## 2026-08-15 — IdP-3 PDP enforce (`rba-idp` → `/risk/evaluate`)

After a successful password verify, `rba-idp` calls `POST /risk/evaluate` and
maps the PDP action onto the login outcome via `outcome_from_action`. Reasons
travel back to the client. This is where RBA re-enters the login path
(ADR-0015).

- `ALLOW` → `AUTHENTICATED`; `REQUIRE_MFA` → `MFA_REQUIRED`;
  `REAUTHENTICATE` → `REAUTH_REQUIRED`; `BLOCK` → `BLOCKED`.
- Invalid credentials / unknown user still skip the PDP (password never leaves
  the IdP). Unknown app remains HTTP 400.
- PDP down or invalid body → HTTP 503 (fail closed; not a fabricated `BLOCK`).
- No session cookie and no `challenge_id` yet — those are IdP-4.

**Next:** IdP-4 — session on `ALLOW`; mock MFA/reauth challenge.

---

## 2026-08-15 — IdP-2 identity store (`rba-idp`)

New `rba-idp` service: local users + one registered application, `POST /login`
password verify (bcrypt). No PDP call, no session, no MFA — those are IdP-3/4.

- DB `rba_idp` added in `rba-infra` init (database-per-service).
- Seed: application `demo-banking-app`, user `demo@example.com` / `demo-password`
  (`usr_demo`) — matches `rba-contracts` login examples.
- Valid credentials → `AUTHENTICATED` + `user_id`. Wrong password / unknown user
  → `INVALID_CREDENTIALS` (HTTP 200). Unknown app → HTTP 400.
- `/mfa/verify`, `/session`, `/logout` are not implemented yet (404).

**Next:** IdP-3 — call `/risk/evaluate`, map action → outcome + reasons.

---

## 2026-08-15 — RBA is the thesis core (ADR-0015)

The Auth0/Authentik-shaped IdP is the **shell**. The thesis is still
**risk-based authentication**: explainable score + action + reasons on a real
login. An IdP that never calls the PDP (skipping IdP-3) would miss the claim.
Admin must show decisions/reasons, not only users.

**Next:** IdP-2 (identity store), then IdP-3 (wire RBA) without skipping.

---

## 2026-08-15 — IdP = thesis-scale Auth0/Authentik (ADR-0014)

Clarified the product metaphor: not a JSON wrapper around the PDP, and not a
carrier-grade IdP. **Like** Auth0/Authentik (users, registered apps, hosted
login, session, MFA, admin). **Unlike** them (no full OIDC/SAML, SCIM, LDAP,
social login). Differentiator remains explainable RBA via the existing PDP.

IdP-2 now includes a seeded **application** (client). Stages otherwise unchanged.

**Next:** IdP-2 — `rba-idp` users + application, password verify only.

---

## 2026-08-15 — IdP staged; IdP-1 contracts only (ADR-0013)

Discarded the Horizon A demo kit. Phase 7 is split into IdP-1…7; this session is
**only IdP-1**.

- ADR-0013: skip demo-kit polish; start the thin IdP one stage at a time.
- `rba-contracts` **0.2.0**: `LoginRequest` / `LoginResponse` / MFA / session.
  PDP `/risk/evaluate` unchanged. Optional risk/session fields stay unset until
  IdP-3/4.
- `status.md` restated as **end product first**: each session advances one IdP
  stage; leftover Phase 4/5/6/0 work is not the current path.

**Next:** IdP-2 — `rba-idp` + `rba_idp` DB, password verify only.

---

## 2026-08-12 — Product target: thin IdP (Oct) on PDP core (demo now)

Locked dual horizon in the **canonical plan** + ADR-0012:

- **Horizon A (~10-day demo):** existing PDP + async plane; no IdP yet.
- **Horizon B (late Oct):** `rba-idp` as PEP (login/session/MFA) calling
  `/risk/evaluate`; admin for users/decisions/policy; **groups/permissions as
  stretch**.
- **Migration rule:** near-term work stays additive — do not put identity in
  `decision-service`; do not treat the demo PEP as the final product shell.

Updated: `plans/development_plan.md` (§0, §1.1, topology, services, Phase 7,
cut order), `plans/status.md`, `decisions/0012-thin-idp-end-product.md`.

**Next:** polish Horizon A for the near demo; IdP only after that.

---

## 2026-08-11 — Phase 4 thin slice: outbox → RabbitMQ → profile/audit

Async plane stood up (ADR-0011):

| Piece | Repo |
|---|---|
| RabbitMQ in shared compose | `rba-infra` |
| Outbox drainer | `rba-event-publisher` |
| Redis profile materialiser | `rba-profile-service` |
| Audit store | `rba-audit-service` |
| `login` on `DecisionMadeEvent` | `rba-contracts` 0.1.1 |

E2E: evaluate with `PROFILE_WRITE_MODE=none` → publisher → Redis profile
`login_count=1` + audit row. Old outbox rows without `login` are skipped by
profile (audited still).

**Next:** remotes/commits for the new repos; then Phase 5 observability or DLQ hardening.

---

## 2026-08-11 — Phase 0 light: `rba-infra` shared Redis/Postgres compose

Moved the local data plane out of `rba-decision-service` into a new `rba-infra`
repo (ADR-0010): one Redis, one Postgres server, init DBs
`rba_decision` / `rba_profile` / `rba_audit`. Service repos consume URLs only.
Full k3d/Tilt/Helm still deferred.

**Next:** run decision-service against this stack end-to-end; then Phase 4 async.

---

## 2026-08-11 — Phase 3 thin slice: `rba-decision-service`

Stood up the request-path PDP against the Phase 2 contracts (ADR-0009):

| Piece | Where |
|---|---|
| `POST /risk/evaluate` | `rba-decision-service` FastAPI |
| ProfileState + Freeman counts + Redis JSON | `rba-features` 0.1.1 |
| Freeman online (`score_event`, JSON export, β=5) | `rba-ml-training` + `artifacts/freeman-0.1.0.json` |
| Policy / reasons / decision+outbox | evaluate service + SQLAlchemy |

Cold-start users correctly get logrisk≈0 (Dirichlet prior = population). Sync
profile writes (`PROFILE_WRITE_MODE=sync`) keep demos working until Phase 4
profile-service. Tests: features 7 passed; decision-service 6 passed / 1 skipped.

**Next:** exercise compose Redis/Postgres day-to-day; then Phase 4 outbox drain.

---

## 2026-08-08 — Phase 2: freeze contracts (`rba-contracts` v0.1.0)

Created the new polyrepo library `rba-contracts` and locked the five Phase-2
artifacts (ADR-0008):

| Contract | Where |
|---|---|
| Feature schema `FeatureVectorV1` (1.0.0) | `schemas/feature-vector.schema.json` + Pydantic |
| Model I/O (`risk_score∈[0,1]`, contributions, artifact sidecar) | `schemas/model-*.schema.json` |
| `POST /risk/evaluate` (PDP) | `openapi/risk-evaluate.yaml` |
| Outbox/bus event `rba.decision.made.v1` | `asyncapi/decision-events.yaml` |
| score→level / level→action policy config | `schemas/policy-config.schema.json` + example YAML |

Key freezes vs the exploratory sketches: probability scale (not 0–100); actions
`ALLOW|REQUIRE_MFA|REAUTHENTICATE|BLOCK`; structured reasons; `event_id` as the
idempotency key end-to-end; Freeman maps `logrisk→[0,1]` via declared
`proba_mapping` (default `logistic_logrisk`); `is_attack_ip` / RTT / absolute geo
kept off the v1 API. `FEATURE_NAMES` here must match `rba-features` (parity test
included). `pytest`: 8 passed.

**Next:** Phase 3 — stand up `rba-decision-service` against these contracts
(Redis profile → features → Freeman inline → policy → decision+outbox).

---

## 2026-08-08 — Freeman calibration (smoothing tune; supervised weighting rejected)

Tried to lift the primary Freeman scorer's weak strict-FPR recall (Step 5: 3/38 @ 1%
FPR) via `ml/calibrate.py` — three levers on the same split, with per-feature LLR
contributions now exposed (`FreemanScorer.contributions_frame`, also future online
explanations). Findings 2026-08-08-freeman-calibration.md.

**Adopted:** lower the Dirichlet smoothing `beta` 10 → 5 (new `FreemanScorer` default).
Label-free, principled (trust a user's short history sooner), improves both ROC-AUC
(0.762 → 0.779) and recall@1%FPR (3/38 → 4/38). **Rejected:** dropping the near-unique
IP (helps operating point but hurts AUC 0.762 → 0.711) and a supervised logistic
per-feature weighting (overfits ~103 train positives → ROC 0.67, recall 0.026). Key
structural note: every variant detects 0 positives in the 0- and 1-2-login buckets —
Freeman's recall is capped by history depth, not feature weighting (logins-to-protection).
Deltas are ±1 detection (38 positives), so the beta change is adopted on principle, not
significance. Refines ADR-0004.

**Next:** Phase 2 — freeze the feature/model/scoring + `/risk/evaluate` contracts.

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
