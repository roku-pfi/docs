# Supervised second opinion, failed-login bands, and the poisoning fix

- Date: 2026-08-19
- Source: `rba-ml-training` — `python -m ml.train --model all`,
  `python -m ml.export_logreg`, `python -m ml.export_freeman`
- Regenerate: see "Reproduction" below.
- Related: [ADR-0027](../decisions/0027-supervised-second-opinion-and-failed-logins.md),
  [ADR-0004](../decisions/0004-modelling-and-label-strategy.md),
  findings [`…step5-baselines.md`](2026-08-08-step5-baselines.md),
  [`…freeman-calibration.md`](2026-08-08-freeman-calibration.md).

> **Supersedes, for any figure that depends on `update_profile`:** the Step-5 and
> calibration findings were measured when failed logins established familiarity.
> Their Freeman numbers still hold (Freeman barely moved); their **LogReg**
> numbers do not. Read this file for the current baseline table.

## 1. The gap this closes

Step 5 measured, on the same chronological split:

| Model | recall @ FPR≈1% | Served? |
|---|---|---|
| freeman | 0.105 (4/38) | **yes** |
| logreg | 0.500 (19/38) | no — "reference baseline" |

The PDP served the weaker model. ADR-0004's justification (no labels needed,
self-explaining, 141 positives in ~31M logins) is sound for choosing the
*primary*, but it did not justify discarding the supervised signal entirely.

## 2. What changed in the features (and why it costs something)

Reporting failed logins to the PDP exposed a poisoning vector: `update_profile`
folded failures into the seen-sets, so an attacker who *fails* from their own
context makes it familiar. Measured directly on the feature vector — 8
legitimate AR logins, then N failed RU attempts, then score an RU login:

| feature | old, 0 failures | old, 4 failures | new, 4 failures |
|---|---|---|---|
| `country_seen_before` | 0 | **1** | 0 |
| `asn_seen_before` | 0 | **1** | 0 |
| `ip_seen_before` | 0 | **1** | 0 |
| `os_seen_before` | 0 | **1** | 0 |
| `device_type_seen_before` | 0 | **1** | 0 |
| `browser_seen_before` | 1 | 1 | 1 |
| `failed_logins_last_24h` | 0 | 4 | 4 |
| `user_login_count` | 8 | 12 | 8 |

**Four wrong passwords flipped five of six novelty signals.** The failure
counter would have *lowered* risk. Fix (ADR-0027): only a successful login
establishes familiarity — a failure appends to `failed_login_ts` and nothing
else.

## 3. Baselines after the change

`python -m ml.train --model all`, same protocol as Step 5 (label-covered window
2020-02-03 → 2020-11-23, 153,093 rows, 141 positives; chronological 70/30 split
at 2020-08-27; test set 45,928 logins / 38 positives).

| Model | ROC-AUC | PR-AUC | Recall @ FPR≈1% | Challenge rate | Size | Latency |
|---|---|---|---|---|---|---|
| **freeman** (primary, β=5) | 0.779 | 0.004 | **0.105** (4/38) | 1.01% | 798 KB | 5.6 µs |
| **logreg** (second opinion) | 0.873 | 0.033 | **0.395** (15/38) | 0.73% | 1.7 KB | 0.1 µs |
| rf | 0.673 | 0.010 | 0.026 (1/38) | 0.01% | 7.0 MB | 1.7 µs |
| lgbm | 0.655 | 0.001 | 0.000 (0/38) | 0.00% | 97 KB | 0.2 µs |

Recall by history depth at the 1%-FPR threshold:

| bucket | n | positives | freeman | logreg |
|---|---|---|---|---|
| 0 (first login) | 8,750 / 12,478 | 8 | 0.00 | 0.00 |
| 1–2 | 10,343 / 10,483 | 9 | 0.00 | 0.78 |
| 3–9 | 14,615 / 12,641 | 15 | 0.27 | 0.40 |
| 10+ | 12,220 / 10,326 | 6 | 0.00 | 0.33 |

(Bucket sizes differ between the two because `user_login_count` no longer counts
failed attempts, so more events sit at depth 0 for the supervised models.)

### Cost of the security fix

| | before (failures established familiarity) | after |
|---|---|---|
| logreg ROC-AUC | 0.880 | 0.873 |
| logreg recall@1%FPR | 0.500 (19/38) | **0.395 (15/38)** |
| freeman ROC-AUC | 0.779 | 0.779 |
| freeman recall@1%FPR | 0.105 (4/38) | 0.105 (4/38) |

Four detections out of 38, to close a vector any attacker can trigger by typing
a wrong password five times. With 38 positives this difference is well inside
the noise band; the poisoning vector is not. Freeman is unaffected — its
per-user counts were already dominated by successful logins.

**Deployed detection is still 3.7× Freeman alone** (0.395 vs 0.105) at a 0.73%
challenge rate.

## 4. What the supervised model actually learned

Coefficients (standardised features, `logreg-0.1.0`):

| feature | coef |
|---|---|
| `country_seen_before` | **−1.899** |
| `ip_seen_before` | −0.858 |
| `asn_seen_before` | −0.735 |
| `os_seen_before` | **+0.820** |
| `browser_seen_before` | **+0.604** |
| `seconds_since_last_login` | +0.428 |
| `device_type_seen_before` | +0.304 |
| `failed_logins_last_24h` | **−0.252** |
| `hour_seen_before` | +0.223 |
| `user_login_count` | +0.195 |

Two results worth defending out loud:

**(a) The takeover signature is "novel place, ordinary device."** The
place signals are strongly protective when familiar; the *device* signals have
**positive** coefficients — a familiar-looking device raises risk. So a wholly
novel login scores *lower* (0.643) than a novel-country-and-network one (0.942)
and does not fire at all. Attackers in this dataset arrive from unseen networks
presenting mainstream OS/browser strings that are "seen before" for most users.
Freeman cannot express this shape: it pushes every categorical through the same
likelihood ratio, so novelty always increases its score. That is precisely the
complementarity that justifies running both.

Firing behaviour at the shipped threshold (0.9028):

| login shape | score | fires |
|---|---|---|
| everything familiar | 0.071 | no |
| novel country | 0.788 | no |
| novel country + ASN | 0.942 | **yes** |
| novel country + ASN + IP | 0.990 | **yes** |
| everything novel | 0.643 | no |

**(b) `failed_logins_last_24h` has a negative coefficient.** More recent
failures correlate with *legitimate* users in this data — people mistype their
own passwords far more often than attackers reach the same account. This is why
ADR-0027 puts the credential-stuffing response in a **deterministic rule** and
not in the model: the population statistic and the correct per-account security
response point in opposite directions. A model trained on this data would learn
to be *more* permissive after repeated failure.

## 5. End-to-end behaviour (compose, PDP + IdP, `PROFILE_WRITE_MODE=sync`)

Same starting profile (8 successful AR logins) in both rows; the only difference
is four failed RU attempts in between; the scored event is an RU login:

| | risk_score | action | reasons |
|---|---|---|---|
| no prior failures | 4.38305e-05 | `REQUIRE_MFA` | `impossible_travel`, 3× `signal_novel` |
| after 4 RU failures | **4.38305e-05** | **`REAUTHENTICATE`** | `failed_login_burst`, `impossible_travel`, 3× `signal_novel` |

The score is byte-identical — failures no longer teach the profile — and the
action escalates on the rule. Continuing to 12 failures yields
`failed_login_lockout` → `BLOCK`.

## 6. Reproduction

From `rba-ml-training/` (venv active, dataset subset present):

```bash
python -m ml.train --model all                       # table in §3
python -m ml.export_freeman --pickle artifacts/step5/freeman.pkl \
    --out artifacts/serving/freeman-0.2.0.json --beta 5.0 --model-version freeman-0.2.0
python -m ml.export_logreg --out artifacts/serving/logreg-0.1.0.json
cp artifacts/serving/{freeman-0.2.0,logreg-0.1.0}.json ../rba-decision-service/artifacts/
```

§4 firing table: `rba-decision-service/tests/test_escalation.py`
(`test_supervised_fires_on_a_novel_place`,
`test_supervised_learned_signature_is_novel_place_familiar_device`).
§5: start compose + PDP + IdP per the root README, then drive
`POST /login` with a wrong password N times before the scored attempt.

## k3d rehearsal — the stuffing scenarios did not demonstrate their mechanism

Measured on the k3d cluster (not compose), against a verified-pristine
`rba:profile:usr_demo` seed (`login_count 98`, `failed 0`, `seen_ips
['203.0.113.10']`), app `demo-banking-app`.

The walkthrough originally ran the credential-stuffing scenarios from
**RU / 12389 / 198.51.100.66 / Linux**. That context alone scores:

| Prior failures | Action | risk_score | Bound by |
|---|---|---|---|
| 0 | `BLOCK` | 0.8920 | Freeman → CRITICAL |
| 4 | `BLOCK` | 0.8920 | Freeman → CRITICAL |

Freeman reaches CRITICAL on novelty alone, so `failed_login_burst` (floor
`REAUTHENTICATE`) was never the binding constraint. Both scenarios collapsed to
`BLOCK`; the lockout one only *appeared* to work. The demo therefore showed no
`REAUTHENTICATE` at all, contradicting its own on-page script.

Fix: run the attack from the user's **own** country and ASN
(`AR / 7303`, novel IP, Linux) — a residential proxy, which is what credential
stuffing looks like in practice. Same context throughout, only the failure
count varies:

| Prior failures | Action | risk_score | Reason |
|---|---|---|---|
| 0 | `ALLOW` | 0.0000 | — |
| 4 | `REAUTHENTICATE` | 0.0000 | `failed_login_burst` |
| 11 | `BLOCK` | 0.0000 | `failed_login_lockout` |

`risk_score` is 0.0000 and `risk_level` stays `LOW` in all three, while the
action walks the ladder. This is the cleanest available evidence for ADR-0027:
Freeman structurally cannot see credential stuffing, the rule can, and the rule
escalates without ever rewriting the score. `rba-demo-banking/tests/test_app.py::test_stuffing_runs_from_the_users_ordinary_context`
pins the context so a future edit cannot quietly re-break it.

### Two operational defects found in the same rehearsal

**`rba-event-publisher` never reconnects.** `BusPublisher.connect()` runs once
at startup (`main.py:41`); there is no reconnect path. After RabbitMQ restarted,
every `publish` raised `ChannelWrongStateError: Channel is closed`, `drain_once`
caught it per row and left `published_at` NULL, so the same rows retried
forever while the log printed `published N outbox row(s)` — `run()` logs
`len(rows)` attempted, not the number that succeeded. The login path keeps
working (the PDP reads Redis directly), but no profile is ever updated again,
so every scenario that depends on accumulated state silently stops working.
Restarting the deployment drains the backlog. **Not yet fixed.**

**12 fired attempts yield 11 recorded failures.** Reproducible, and identical
whether driven through the demo app or fired straight at `POST /login`; it
settles within 3 s and does not converge to 12. Not root-caused. It does not
break the scenario (11 ≥ the lockout threshold of 10) but it does mean the
lockout reason reads "11 failed logins", and the margin is thinner than the
scenario's 12 attempts suggest.

### Resetting between rehearsals

The seed is `SET NX`, so `DEL` + restarting `profile-service` re-seeds — but an
in-flight decision event can land *after* the reseed and re-pollute the blob.
This silently corrupted three measurements before it was caught (they showed
`score 0.5000` with `low_history`, i.e. an empty profile).
`rba-infra/scripts/reset-demo.sh` retries until the blob matches the pristine
seed exactly and prints it. Do not skip the check.

### Freeman's novelty ceiling — why `new_country` is not a Freeman catch

Freeman's per-signal term is `log p_global(v) − log p_user(v)` with
`p_user = (count + β·p_global) / (total + β)`. For a value the user has never
had, `count = 0`, so the term collapses to `log((total + β)/β)` — independent of
which signal it is, and independent of how rare the value is. On the 98-login
demo seed (β=5) that ceiling is `log(103/5) = 3.025`, and every novel signal
contributes exactly that. Familiar values have no such ceiling: they scored
−7.2 to −12.4 each on the same profile.

| Case | Sum of LLRs | risk_score |
|---|---|---|
| Cold profile (no history) | 0.000 | 0.5000 |
| Seeded, usual home login | −56.899 | 0.0000 |
| Seeded, new country + new ASN (DE / 3320) | −31.368 | 0.0000 |
| Seeded, new country + ASN + IP + OS | −1.768 | 0.1458 |

Two consequences worth stating in the report:

1. **`risk_score = 0.5` means "no information", not "medium risk".** With an
   empty profile every term is exactly 0 and the sigmoid returns 0.5 — the
   scorer abstains. Under `demo-banking-app`'s bands that lands in HIGH, so a
   first-ever login is challenged because nothing is known about it. That is a
   defensible policy stance, but it is a policy stance, not a model output.
2. **Novelty saturates while familiarity accumulates.** A well-established user
   cannot be pushed over the threshold by novelty alone; it takes roughly four
   simultaneously-novel signals before the sum approaches zero. This is the
   structural reason the supervised second opinion earns its place: on the same
   DE / 3320 event Freeman scores 0.0000 (`ALLOW`) while LogReg scores 0.9823
   against its 0.9028 threshold and raises the floor to `REQUIRE_MFA`.

The walkthrough previously advertised `new_country` as "MFA from Freeman +
policy (unseen country)". It never was; the copy has been corrected.

## Caveats

- 38 test positives. Every recall figure here has a wide interval; read the
  0.395 vs 0.105 gap as a direction, not a tuned number.
- The supervised threshold is calibrated on this split and ships in the
  artifact. Any retrain must re-export it and update this file.
- The burst/lockout thresholds (3 / 10) are not tuned against data — they are
  a policy stance about how many wrong passwords should cost a step-up. They
  are service config, so they are not per-application yet.
