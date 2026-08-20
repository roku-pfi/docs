# ADR-0027: Supervised second opinion, failed-login bands, and success-only familiarity

- Status: Accepted
- Date: 2026-08-19
- Refines [ADR-0004](0004-modelling-and-label-strategy.md) (Freeman stays the
  primary scorer; the supervised baseline stops being *only* a reference number)
  and generalises the escalate pattern from [ADR-0022](0022-product-demo-over-gitops.md)
  into one action ladder. Does not change the PDP/PEP split
  ([ADR-0012](0012-thin-idp-end-product.md), [ADR-0015](0015-rba-is-the-thesis-core.md))
  or the evaluation protocol ([ADR-0007](0007-evaluation-protocol.md)).
- Numbers: [`findings/2026-08-19-supervised-escalation-and-failed-logins.md`](../findings/2026-08-19-supervised-escalation-and-failed-logins.md)

## Context

Three problems, all visible in what was already shipped.

**1. We served the weakest model at the operating point.** Step 5 measured
Freeman at recall@1%FPR **0.105** and a plain LogisticRegression at **0.500** on
the same chronological split. The PDP served Freeman alone. ADR-0004's
reasoning is sound — Freeman needs no labels and explains itself, and 141
positives in ~31M logins cannot support a supervised primary — but the shipped
system left a 5× detection gap on the table, and the thesis had a table where
the deployed model is the worst one in it. That is the first question a
committee asks.

**2. `failed_logins_last_24h` was structurally always 0.** The feature exists in
`rba-features`, the reason code existed in the PDP, and nothing ever produced a
non-zero value: the IdP returned `INVALID_CREDENTIALS` on a wrong password
without telling the PDP. Credential stuffing — the threat the proposal leads
with — was invisible to the decision. The demo showed only `ALLOW` and
`REQUIRE_MFA`; `REAUTHENTICATE` and `BLOCK` never appeared on stage.

**3. Reporting failures naively creates a profile-poisoning vector.** Once the
IdP reports failed attempts, `update_profile` folds them into the seen-sets and
Freeman counts like any other event. An attacker who submits wrong passwords
from their own IP/country/device thereby *teaches the profile that their context
is normal*. Measured: four failed attempts flipped five of six novelty signals
from novel to seen-before, so the failure counter would lower risk instead of
raising it.

## Decision

**1. Freeman remains the primary score; a supervised model may only escalate.**
`rba-decision-service` loads a second JSON artifact (`logreg-0.1.0.json`:
mean/scale/coef/intercept plus a baked operating point) and scores the same
feature vector with one dot product — no sklearn, no pickle, no extra process,
no network hop. If it scores above its offline-calibrated 1%-FPR threshold, the
PDP raises the action to at least `REQUIRE_MFA` and emits a
`supervised_second_opinion` reason.

`risk_score` in the response stays Freeman's number, always. The supervised
model never rewrites it and never *lowers* an action. So ADR-0004 stands:
the label-free, self-explaining scorer is still what the level and the reason
trace are built from; the supervised model is a second opinion with a
deliberately conservative operating point (0.73% challenge rate).

The decision threshold is calibrated offline and **ships inside the artifact**.
Serving never re-derives it.

**2. Failed logins reach the PDP, and drive two deterministic bands.**
`rba-idp` reports a wrong password to `/risk/evaluate` with
`login_successful=false`, then answers exactly as before. The reporting call is
best-effort: a PDP outage must not turn a wrong password into a 503. The
returned decision is discarded — a failed attempt is not an authentication and
there is nothing to enforce.

`failed_logins_last_24h` then drives an escalation with two bands:

| Band | Default | Action floor | Reason code |
|---|---|---|---|
| burst | ≥ 3 in 24h | `REAUTHENTICATE` | `failed_login_burst` |
| lockout | ≥ 10 in 24h | `BLOCK` | `failed_login_lockout` |

This is a **rule**, not a model input, and deliberately so — see the finding:
after (3), the fitted supervised coefficient on `failed_logins_last_24h` is
*negative*, because in this dataset most failures are legitimate users
mistyping. The population statistic and the per-account security response point
in opposite directions, so the response belongs in policy where it is
inspectable and tunable, not buried in a coefficient.

**3. Only a successful login establishes familiarity.** In `rba-features`,
`update_profile` on a failed event appends to `failed_login_ts` and does nothing
else: no seen-set, no Freeman count, no `last_login_ts`, no `login_count`. This
is the same rule the travel anchors already applied, now applied consistently.
Because it lives in the shared library, offline replay and online serving move
together and parity is preserved.

**4. One action ladder.** `ALLOW < REQUIRE_MFA < REAUTHENTICATE < BLOCK`.
Every rule and second opinion names a floor; `services/escalate.escalate` takes
the more severe. Travel/VPN, the failed-login bands, and the supervised opinion
all go through it. Nothing can lower an action.

## Consequences

- Positive: the deployed system now detects ~3.7× what Freeman alone did at the
  same operating point (0.395 vs 0.105), while the explanation and the score
  stay Freeman's. Credential stuffing is a real, demonstrable signal. The
  walkthrough exercises all four actions. Repeated failure can no longer be used
  to normalise an attacker's context.
- Negative: **the security fix costs detection.** Excluding failures from the
  profile drops supervised recall@1%FPR from 0.500 to 0.395 and shifts the
  history-depth buckets (more events now sit at depth 0). Numbers in the Step-5
  and calibration findings describe the *previous* feature semantics and are
  superseded for any figure that depends on `update_profile`. The trade is
  accepted: a trivially exploitable poisoning vector is worse than 4 fewer
  detections out of 38.
- The Freeman serving artifact was refit under the new semantics and is now
  **`freeman-0.2.0.json`**. Freeman's own metrics barely moved (ROC-AUC 0.779,
  recall 0.105).
- **Migration:** `ProfileState` blobs materialised in Redis before this change
  contain failure-derived familiarity. They are not wrong-shaped, just
  optimistic, and decay only as new successes accumulate. Rebuild the demo seed
  (and any profile used for a measurement) rather than trusting an old blob.
- Two new knobs, both with defaults: `FAILED_LOGIN_LOCKOUT_THRESHOLD` (10) and
  `SUPERVISED_ESCALATION_ENABLED` (true). `REPORT_FAILED_LOGINS` (true) on the
  IdP turns reporting off if a measurement needs the old behaviour.
- Follow-up: the burst/lockout thresholds are service config, not policy YAML,
  so they are not per-application yet. If a low-sensitivity app should tolerate
  more failures, they move into `PolicyConfig` — a contracts change.
