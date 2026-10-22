# D-2 — Policy comparison on the held-out logins

- Date: 2026-10-22
- Source: `rba-ml-training/ml/policy_compare.py` → `reports/policy_compare/`.
  Numbers: [`data/2026-10-22-policy-comparison.json`](data/2026-10-22-policy-comparison.json).
- Regenerate: from `rba-ml-training/`, `python -m ml.policy_compare` (`--fresh` to
  replay again; ~1 min).

## Method

The Step 5 test split (label window to 2020-11-23, chronological 70/30 cut at
2020-08-27 20:45) replayed through the **production decision code**: the deployed
`freeman-0.2.0` and `logreg-0.1.0` artifacts, `apply_policy` with the IdP's starter
policy (LOW ≤ 0.30 → allow, ≤ 0.60 → passkey, ≤ 0.80 → re-authenticate, above →
block), and the decision service's rules (failed-login burst/lockout, impossible
travel and VPN, the supervised second opinion), each only raising the action. Each
login is scored against its user's history up to that moment, then folded in.

Population: the test split's **successful** logins, the ones a policy acts on:
**34,193 legitimate** and **38 account takeovers** (all attackers held the right
password). A takeover counts as stopped by any non-allow action: a step-up stops an
attacker without the user's passkey. 95% intervals: Wilson per login; a user-level
bootstrap (2,000 resamples) as a check, since one user's logins are not independent.
At 0/38 and 38/38 the bootstrap degenerates to a point; cite Wilson there.

## Results

| Policy | Legit friction | 95% CI | Takeovers stopped | 95% CI | Legit blocked |
|---|---|---|---|---|---|
| Password only | 0.00 % | [0.00, 0.01] | 0/38 (0 %) | [0, 9.2] | 0.00 % |
| MFA always | 100.00 % | [99.99, 100] | 38/38 (100 %) | [90.8, 100] | 0.00 % |
| **Adaptive (engine, starter policy)** | **38.06 %** | [37.54, 38.57] | **32/38 (84.2 %)** | [69.6, 92.6] | **6.21 %** |
| **Adaptive, critical steps up instead of blocking** | **38.06 %** | [37.54, 38.57] | **32/38 (84.2 %)** | [69.6, 92.6] | **0.02 %** |
| **Adaptive, new default (ADR-0040): LOW ≤ 0.40, critical steps up** | **36.02 %** | [35.51, 36.53] | **32/38 (84.2 %)** | [69.6, 92.6] | **0.02 %** |
| Freeman only (ablation) | 36.72 % | [36.21, 37.23] | 31/38 (81.6 %) | [66.6, 90.8] | 6.19 % |

## Findings

1. **The adaptive engine stops 84 % of takeovers (CI 70–93 %) while challenging
   38 % of legitimate logins**, against 100 % for MFA always. With 38 positives
   the interval is wide; the claim is "most", not a precise rate.
2. **Most of the friction is the cold start.** A user's first login has no history,
   so every one is challenged (6,613 logins, 100 %). For returning users, under the
   adopted default, friction is **20.7 %** and the engine stops **24 of 30**
   takeovers.

   | Prior successful logins | Legit logins | Friction | Takeovers | Stopped |
   |---|---|---|---|---|
   | 0 | 6,613 | 100.0 % | 8 | 8 |
   | 1–2 | 7,523 | 28.4 % | 9 | 9 |
   | 3–9 | 10,563 | 23.1 % | 15 | 12 |
   | 10+ | 9,494 | 11.9 % | 6 | 3 |
3. **Blocking on the score is all cost, no benefit.** The starter policy's
   "critical → block" band blocks **6.21 %** of legitimate logins. Stepping up
   instead (re-authenticate with the passkey) stops exactly the same 32 takeovers
   and blocks **0.02 %** (only the failed-password lockout rule still blocks). For
   a score with this precision, a block should come from a rule, not from the score.
4. **The rules and the second opinion add one takeover** (31 → 32) for 1.3 points
   of friction. Small on this dataset; the travel and burst rules matter more for the
   attacks the dataset under-represents (credential stuffing, travel from abroad).
5. **The LOW band's upper limit is flat up to 0.40, then falls off a cliff**: friction
   41 → 36 % from 0.10 to 0.40 at the same 84 % stopped; at 0.50 friction drops to
   15 % but stopped falls to 63 %. Freeman's scores cluster, so the threshold
   is a step, not a slope. 0.40 is the better default on this data.

   | LOW ≤ | Friction | Stopped |
   |---|---|---|
   | 0.10 | 41.0 % | 84.2 % |
   | 0.30 | 38.1 % | 84.2 % |
   | 0.40 | 36.0 % | 84.2 % |
   | 0.50 | 15.3 % | 63.2 % |

## Outcome

Adopted as the default in every seeded policy and in the starter policy
([ADR-0040](../decisions/0040-scores-step-up-rules-block.md)): the same detection
as before, 2 points less friction, and 0.02 % of legitimate logins blocked instead
of 6.21 %.

## Caveats

- 38 positives: one takeover is 2.6 points. All rates on the takeover side are
  coarse.
- The dataset (Wiefling et al.) is a single online service's logins; travel and
  VPN signals are those of its population.
- "Stopped" assumes the attacker cannot pass the step-up (no access to the
  user's passkey). Session theft after login is out of scope.
