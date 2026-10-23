# Freeman's hierarchical smoothing, and why it needs a global root

- Date: 2026-10-23
- Source: Freeman, Jain, Dürmuth, Biggio, Giacinto, *Who Are You? A Statistical
  Approach to Measuring User Authenticity*, NDSS 2016 (Sect. II-C, V);
  `rba-ml-training` — `python -m ml.hierarchical_eval` (→ `reports/hierarchical.md`),
  `python -m ml.hierarchical_robustness`; model in `ml/models/hierarchical.py`.
- Related: [Freeman in the cloud](2026-10-23-freeman-in-the-cloud.md),
  [calibration](2026-08-08-freeman-calibration.md), [Step 5](2026-08-08-step5-baselines.md).

## What the paper does that we did not

| | Paper | freeman-0.3.0 |
|---|---|---|
| Features | 2: IP, user agent | 7 flat: ip, asn, country, device, os, browser, hour |
| Correlated parts | nested hierarchies (IP → org → AS → country → world; UA → browser → OS → app → world) | each level independent evidence: a new network counts three times |
| Unseen values | Eq. 8 per level with μ_h = \|h\| unseen leaves; levels combined by backoff or linear interpolation (λ learned on held-out legitimate logins) | Dirichlet toward the global value |
| Paper's result (LinkedIn) | IP + UA, interpolation, μ = \|h\|: AUC 0.913 (0.70 with μ = 1) | — |

## Results (Step 5 split, 38 test takeovers)

Network = world → country → ASN → IP; agent = world → device → +OS → +browser
(families). Tables from the training period, users past-only.

| variant | ROC-AUC | recall@1%FPR | TPR@10%FPR |
|---|---|---|---|
| flat freeman-0.3.0 | 0.807 | 0.132 (5/38) | 0.474 |
| paper as published (network + agent) | 0.763 | 0.000 (0/38) | 0.000 |
| paper + hour / + p(u) term | 0.768 / 0.772 | 0 / 1 of 38 | 0.000 / 0.132 |
| **paper's hierarchy, user rooted at the global estimate** | **0.887** | **0.579 (22/38)** | **0.579** |
| same, uniform λ (nothing learned) | 0.913 | 0.579 (22/38) | 0.579 |

## Why the paper's form fails here, and the fix

The paper smooths a user's hierarchy within the user's own counts: a value new
to the user gets 1/(N_u + M_u) at the world level, whatever its global rarity.
A value that is both new to the user and rare globally then scores
log(tiny) − log(1/(N_u+M_u)): "safe". 68% of the test takeovers' IPs were never
seen in the training period (54% of legitimate logins'), so the paper's model
calls most attacks safe. Its LinkedIn attackers were botnets on globally common
IPs, where the same formula works; our cloud attacker (a fresh AWS IP in Tokyo)
is the rare case.

Rooting the user's hierarchy at the population (the world level of
p(x | u, L) is p(x)) is Freeman's own Dirichlet idea applied at the top: a value
new at every level costs −log λ₀, bounded and independent of rarity; a new IP
inside the user's usual ASN keeps the ASN level's probability, so a network
change counts once.

## Robustness

| check | ROC-AUC | recall@1%FPR |
|---|---|---|
| λ on held-out legitimate logins | 0.887 | 22/38 |
| λ on all held-out logins (no labels) | 0.887 | 22/38 |
| λ held-out = last 20% of training | 0.886 | 22/38 |
| uniform λ | 0.913 | 22/38 |

Recall@1%FPR, rooted − flat: **+0.44, 95% CI [+0.28, +0.60]** (bootstrap over
users, 1000 resamples).

By history depth, at each model's own 1%-FPR threshold:

| prior logins | flat | rooted |
|---|---|---|
| 0 | 0/8 | 0/8 |
| 1–2 | 0/9 | **9/9** |
| 3–9 | 4/15 | 11/15 |
| 10+ | 1/6 | 2/6 |

The "logins-to-protection" ceiling of the calibration finding was a property of
the flat model: a new country now weighs about +5 from the second login.

## Against Wiefling et al.'s attacker models (the dataset's own paper)

The dataset (Wiefling, Jørgensen, Thunem, Lo Iacono, Zenodo 6782156; ACM TOPS
2022) was synthesized to preserve Freeman's results with exactly the paper's
hierarchies (IP: address, ASN, country; UA: string, OS, browser, device), with
IP ⊂ ASN ⊂ country by construction. Its paper evaluates RBA against simulated
attackers, all with the password: naive (random or most frequent attack IPs and
UAs), VPN (attack IP in the victim's country, popular UA), targeted (frequent
attack IP in the victim's country, the victim's UA). `python -m ml.attacker_eval`
replays 4,000 probes per type against each victim's state at a test-period
login (scored, never learned), using the dataset's `is_attack_ip` attempts.

Attackers caught at 5% / 10% / 20% / 30% of legitimate logins challenged:

| attacker | flat freeman-0.3.0 | rooted hierarchy, uniform λ | chain rule, β per level fitted |
|---|---|---|---|
| naive (random) | 0.65 / 0.80 / 0.95 / 0.99 | 0.97 / 0.99 / 0.99 / 0.99 | 0.99 / 0.99 / 1.00 / 1.00 |
| naive (resourced) | 0.52 / 0.66 / 0.80 / 0.88 | **0.95 / 0.98 / 0.99 / 1.00** | 0.86 / 0.91 / 0.97 / 0.99 |
| VPN | 0.29 / 0.42 / 0.62 / 0.76 | **0.72 / 0.83 / 0.91 / 0.95** | 0.51 / 0.70 / 0.88 / 0.95 |
| targeted | 0.05 / 0.11 / 0.25 / 0.47 | **0.52 / 0.68 / 0.80 / 0.86** | 0.06 / 0.18 / 0.49 / 0.73 |
| labelled takeovers | 0.32 / 0.42 / 0.79 / 0.84 | **0.87 / 0.87 / 0.97 / 0.97** | 0.58 / 0.61 / 0.82 / 0.82 |

Adding `hour` lowers every row (it is dropped from the score; `hour_seen_before`
stays for the supervised second opinion). λ fitted by likelihood is never
better than uniform.

**Chain-rule Freeman** (p(country)·p(ASN|country)·p(IP|ASN), flat Freeman's
Dirichlet at each level, β per level fitted on held-out legitimate logins:
country 0.046, ASN 0.83, IP 1.78; device 1.0, OS 0.38, browser 0.46) counts each
novelty once and makes it independent of the new value's rarity. It wins only
in the extreme tail (blocking 99% of targeted attackers: 44% of legitimate
logins challenged, against 85% for the rooted hierarchy). Rejected: making a
novelty independent of rarity throws away Freeman's attacker model. Among
values new to the user, the rooted hierarchy rates those common in the
population (shared attack IPs, popular user agents) as more attacker-like, and
that is what separates the targeted attacker in the victim's own country.

## What was adopted: the hybrid (freeman-0.4.0, ADR-0043)

Projected on the cloud's situation (Martín alone in a fresh deployment), the
fully rooted model rated a common Windows + Chrome attacker from Tokyo LOW
(0.07 → 0.50 over 1 → 15 logins): the agent hierarchy compares a new browser
with its popularity in the population, and **this dataset pseudonymizes values
per unique value** (Zenodo record; Wiefling et al. Sect. 11), so the training
data's "Windows" is a placeholder with 31 logins and popularity by name has no
real-world meaning. The network part does not have that problem in the demo (a
new country costs the full log 4 whatever the population says).

**Hybrid: rooted hierarchy for the network, flat Freeman for device / OS /
browser.**

| | ROC-AUC | recall@1%FPR | TPR@10%FPR |
|---|---|---|---|
| flat freeman-0.3.0 | 0.807 | 0.132 | 0.474 |
| rooted hierarchy (both) | 0.887 | 0.579 | 0.579 |
| **hybrid (freeman-0.4.0)** | **0.889** | **0.526** | **0.526** |

Attackers caught at 5% / 10% / 20% / 30% challenged, hybrid: naive random
0.99 / 1.00 / 1.00 / 1.00; resourced 0.96 / 0.99 / 1.00 / 1.00; VPN
0.68 / 0.81 / 0.92 / 0.96; targeted 0.40 / 0.66 / 0.80 / 0.86; takeovers
0.84 / 0.84 / 0.92 / 0.92.

Policy comparison (starter defaults, ADR-0040), freeman-0.3.0 → 0.4.0:
legitimate friction **35.9% → 21.4%**, takeovers stopped 32/38 → 31/38,
legitimate blocks 0.02%; friction for users with 1–2 / 3–9 / 10+ prior logins
25.9 / 21.3 / 15.5% → **2.7 / 2.1 / 3.0%**. First logins stay challenged.

Demo projection (Roku Bank bands), Martín with 1 / 3 / 7 / 15 prior logins:
Tokyo with Windows + Chrome **0.76 / 0.75 / 0.78 / 0.85 (CRITICAL)**; Tokyo
spoofing his macOS browser 0.64 / 0.47 / 0.37 / 0.35; usual login ≤ 0.02;
new wifi ≤ 0.03; his phone 0.03 → 0.94 as history grows. The demo attackers now
claim Windows + Chrome, as real kits and Wiefling's VPN attacker do.

## Limits

- 38 positives; the interval above is wide even though it excludes zero.
- The RBA dataset is partly synthetic (Wiefling et al.); 72% of its takeovers
  change IP, ASN and country at once, which a hierarchy rewards. Targeted
  attackers who match the victim's country are the evasion case the paper
  studies (Sect. V-E) and are not in these labels.
- λ fitted by likelihood optimises density estimation, not detection; uniform λ
  ranked better. Choosing between them is a modelling decision, not settled here.
- With uniform λ, each hierarchy contributes at most log 4 (+1.39), so the
  score's range is narrower than flat Freeman's; bands are recalibrated on the
  policy comparison, not carried over.
- The simulated attackers reuse the dataset's attack IPs, which are frequent in
  the population by construction (the synthesis kept attack IPs as attack IPs);
  an attacker on a fresh, unique IP gets less from the population term.
