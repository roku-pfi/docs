# 42. The profile learns only completed logins; Freeman scores against the deployment's population

- Status: Accepted
- Date: 2026-10-23
- Related: [ADR-0004](0004-modelling-and-label-strategy.md) (Freeman primary),
  [ADR-0027](0027-supervised-second-opinion-and-failed-logins.md) (only a successful
  login establishes familiarity), [ADR-0040](0040-scores-step-up-rules-block.md),
  finding [2026-10-23](../findings/2026-10-23-freeman-in-the-cloud.md).

## Context

The first rehearsal on AWS (D-3) showed the engine scoring every login LOW,
including an attacker with Martín's password logging in from Tokyo. The
hosting-network rule still asked for the passkey, so the outcome was right and
the e2e test, which accepted either the rule or a high score, passed. Three
separate defects were behind it:

1. **Profile poisoning.** "Successful" meant "correct password". A login that
   stopped at the passkey taught the profile its country, network, IP and OS,
   so one abandoned Tokyo attempt made Japan familiar for the next one.
2. **Train/serve skew in browser and OS.** Training saw `Chrome 73.0.3654` and
   `Mac OS X 10.14.6`; the hosted login sends `Chrome` and `macOS`. Every live
   value was unknown to the model, and Freeman reads an unknown value as one
   almost nobody uses: a familiar Chrome added −11.7 to the log-risk, against
   at most +0.9 for a new country.
3. **A population prior from another service.** Freeman's attacker model is the
   population's login distribution. Ours was a 2020 service in Norway, where
   Argentina and 22:00 local time are rare, so every familiar value of an
   Argentine user looked like overwhelming evidence that it was him.

## Decision

- **A login teaches the profile only if it ended in a session.** At once for an
  enforced ALLOW; after the step-up for REQUIRE_MFA / REAUTHENTICATE, when the
  IdP calls the PDP's new `POST /risk/confirm` and the PDP emits
  `rba.login.confirmed.v1` (contracts 0.15.0) through its outbox; never for
  BLOCK. A wrong password still only adds to the failure count. The rule lives
  in `rba_features.features.learn_at_decision`, shared by profile-service and
  the PDP's sync mode.
- **Browser and OS are scored as families** (`rba_features.canonical`), on both
  sides: training maps its strings to families before fitting, and every live
  event is mapped before features and scoring. Idempotent, so stored profiles
  stay valid.
- **Freeman's population is the deployment's own**, with the training data as a
  Dirichlet prior worth κ = 100 logins:
  `p_global(v) = (N_dep(v) + κ·p_train(v)) / (N_dep + κ)`. Only learned logins
  are counted (Redis hash `rba:population`, written by profile-service, read by
  the PDP in one HMGET). If the counts cannot be read, the PDP scores against
  the training population alone instead of falling back.
- Artifacts **freeman-0.3.0** (families, κ = 100) and **logreg-0.2.0**
  (retrained on family-level features) replace freeman-0.2.0 / logreg-0.1.0.
- The console marks a verdict set by a rule ("Por regla", the rule on hover), so
  a BLOCK next to a LOW score reads as what it is.

## Consequences

- Offline, on the step-5 split: Freeman ROC-AUC 0.779 → 0.807, recall@1%FPR
  0.105 → 0.132; LogReg recall@1%FPR 0.395 → 0.474 (ROC-AUC 0.873 → 0.865).
- Under a simulated population shift, the adaptive prior removes most of the
  offset that made fixed bands stop firing (genuine median log-risk −20.3 →
  −2.1 against −7.3 at the source). With 16 positives, recall at fixed
  thresholds cannot be told apart; that limit is stated in the finding.
- A tenant with very few logins scores mostly against the training population
  until it has seen about κ logins of its own. The population is shared by all
  tenants (a per-tenant model stays parked).
- A step-up confirmation that is lost (PDP down between the two calls) only
  means one login is not learned.
