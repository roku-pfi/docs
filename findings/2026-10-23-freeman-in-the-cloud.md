# Freeman in the cloud: why every login scored LOW, and the fix

- Date: 2026-10-23
- Source: AWS rehearsal (D-3), the decision store read through the audit API;
  `rba-ml-training` — `python -m ml.population_shift`,
  `python -m ml.train --model all --out artifacts/step5-families --report reports/step5/metrics-families.md`.
- Related: [ADR-0042](../decisions/0042-learn-from-completed-logins-and-deployment-population.md),
  [Freeman calibration](2026-08-08-freeman-calibration.md), [Step 5](2026-08-08-step5-baselines.md).

## What the cloud showed

Martín's decisions in Roku Bank after the presenter scenes: score **0.00**
on every login after his first, including the Tokyo attacker (JP, AS16509, a
new IP). The passkey was still asked for, because the hosting-network rule
fired, so the outcome looked right and the e2e test, which accepted the rule
*or* a high score, passed.

Per-signal contributions of the first Tokyo attempt:

| signal | contribution | why |
|---|---|---|
| browser `Chrome` (familiar) | **−11.7** | not in the training vocabulary (`Chrome 73.0.3654`…): p_global ≈ 4.7·10⁻⁶ |
| hour (familiar) | −3.4 | 01:00 UTC is rare in the training population (Norway) |
| IP, ASN, country (new) | +0.9 each | capped at log((t+β)/β) with Martín's short history |

One login later, Japan, AS16509, the attacker's IP and Linux were all
"familiar": the attempt that stopped at the passkey had been learned.

## Three defects

1. **Profile poisoning**: "successful" meant "correct password", whatever
   happened at the step-up.
2. **Train/serve skew**: browser and OS reached the model as families online
   and as full version strings in training. Every live value was out of
   vocabulary, which Freeman reads as "almost nobody uses this".
3. **Population shift**: Freeman's attacker model is the population's login
   distribution, and ours came from a different service and country.

## Measurements (Step 5 split)

**A. Browser / OS as families**: better on the dataset itself.

| variant | ROC-AUC | recall@1%FPR |
|---|---|---|
| raw strings (freeman-0.2.0) | 0.779 | 0.105 (4/38) |
| families (freeman-0.3.0) | **0.807** | **0.132 (5/38)** |

LogReg retrained on family-level features (logreg-0.2.0): recall@1%FPR
0.395 → **0.474**, ROC-AUC 0.873 → 0.865. The two reference baselines that are
not served dropped (RF 0.673 → 0.609, LGBM 0.655 → 0.603 ROC-AUC).

**B. Deployment-learned population, same population**: costs almost nothing.

| κ | ROC-AUC | recall@1%FPR |
|---|---|---|
| training only | 0.807 | 0.132 |
| 1000 | 0.804 | 0.132 |
| 100 | 0.802 | 0.132 |

**C. Population shift**: prior fitted on Norwegian users, thresholds set on
their test logins, applied to everyone else (the deployment's situation).

| κ | genuine median log-risk, source → target | target FPR / recall at the source's 1% · 5% · 20% thresholds |
|---|---|---|
| training only | −6.7 → **−20.3** | 0.001/0.00 · 0.003/0.06 · 0.008/0.25 |
| 100 | −7.3 → **−2.1** | 0.002/0.00 · 0.004/0.06 · 0.061/0.31 |

With the training prior only, genuine logins from the unseen population sit
~14 nats lower, so fixed bands stop firing on them (target FPR 0.8% at a
threshold that flags 20% at the source). The deployment-learned prior removes
most of that offset (it overshoots by ~5 nats).

**Policy comparison re-run** (`ml.policy_compare`, same replay as
[2026-10-22](2026-10-22-policy-comparison.md); data in
`data/2026-10-23-policy-comparison.json`, now served to the console's
Efectividad page): the adopted default keeps 32/38 takeovers stopped, legitimate
friction 36.0% → 35.9%, legitimate blocks 0.02%. Freeman alone stops 30/38
(was 31/38; one detection, within noise). ADR-0040's conclusions stand.

## Demo check (families + κ = 100)

Simulated with the exported artifact: Martín with 10 logins in a deployment of
30, all from his laptop.

| login | before | after |
|---|---|---|
| usual | LOW | LOW (0.002) |
| Tokyo attacker, Linux UA | LOW (0.00) | **CRITICAL (0.92)** |
| Tokyo attacker, spoofed macOS UA | LOW | **HIGH (0.66)** |
| his phone (new device and ISP) | — | CRITICAL |
| new wifi in the same city | — | LOW (0.17) |

## Verified on AWS (after the redeploy)

Martín: 7 passkey logins, then the presenter's Tokyo, burst and lockout scenes
(25 decisions). Every decision was replayed through freeman-0.3.0 offline
(`rba-decision-service/scripts/replay_decisions.py`, decisions from the audit
API): **largest difference from production 0.00e+00**, per signal and per
score. The profile grew 0 → 7 with the confirmed passkey logins and stayed at 7
through 18 attacker attempts; Japan and AS16509 stayed new (+0.88 each).

Tokyo scored 0.647 (HIGH in Roku Bank's bands, 0.20 / 0.45 / 0.70). Projected
from the same state:

| Martín's prior logins | Tokyo, Linux UA | Tokyo, his macOS UA | usual login |
|---|---|---|---|
| 7 | 0.647 | 0.235 | 0.00003 |
| 10 | 0.826 | 0.383 | 0.00005 |
| 15 | 0.946 | 0.636 | 0.0001 |
| 20 | 0.980 | 0.806 | 0.0003 |
| 30 | 0.996 | 0.943 | 0.0008 |

A new value adds log((t+β)/β) whatever its rarity (Freeman's formula), so
protection grows with history: the logins-to-protection ceiling, measured live.

## Limits

- 38 test positives (16 in the shifted target): recall differences of one or
  two detections are noise. A is adopted because it removes a skew and ranks
  better; C because it removes a measured offset, not for a recall gain.
- The shift in C is simulated within one dataset; the cloud's (2020 Norway →
  2026 Argentina) is larger.
- κ = 100 is a judgement: the deployment outweighs the training data after
  about 100 learned logins. B shows the ranking barely depends on it.
- With very little history, Freeman cannot make a new country weigh much:
  log((t+β)/β) is +0.9 at t = 8. That is the logins-to-protection ceiling from
  the calibration finding, now visible live.
