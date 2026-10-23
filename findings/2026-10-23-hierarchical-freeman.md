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

## Limits

- 38 positives; the interval above is wide even though it excludes zero.
- The RBA dataset is partly synthetic (Wiefling et al.); 72% of its takeovers
  change IP, ASN and country at once, which a hierarchy rewards. Targeted
  attackers who match the victim's country are the evasion case the paper
  studies (Sect. V-E) and are not in these labels.
- λ fitted by likelihood optimises density estimation, not detection; uniform λ
  ranked better. Choosing between them is a modelling decision, not settled here.
- Offline only. Online it needs per-user prefix counts (country, country+ASN,
  …) and distinct-leaf counts, the deployment-population blend of ADR-0042 at
  each level, and new score bands (the log-risk scale changes).
