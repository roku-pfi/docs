# 43. freeman-0.4.0: the network as Freeman's hierarchy rooted at the population

- Status: Accepted
- Date: 2026-10-23
- Supersedes the scorer of [ADR-0042](0042-learn-from-completed-logins-and-deployment-population.md)
  (its learning rule, browser/OS families and deployment population stay).
- Evidence: finding [2026-10-23-hierarchical-freeman](../findings/2026-10-23-hierarchical-freeman.md).

## Context

Freeman et al. (NDSS 2016) score two features, IP and user agent, each as a
nested hierarchy smoothed level by level. We scored seven flat features, so a
new network counted three times (IP, ASN, country) and a novelty's weight grew
only with history. Implemented as published, the paper's model scored 0/38
takeovers at 1% FPR on our split: it smooths a user within their own counts,
so a value new to the user and rare in the population looks safe, and 68% of
the takeover IPs had never been seen in training. Rooting the user's hierarchy
at the population fixes that and leads every comparison offline, including
Wiefling et al.'s simulated attackers (the dataset's own paper).

For device / OS / browser the rooted hierarchy leans on how popular a browser is
in the population. The dataset pseudonymizes values per unique value, so that
popularity has no real-world meaning, and in a small deployment the model rated
a common Windows + Chrome attacker from Tokyo LOW.

## Decision

- **freeman-0.4.0 is a hybrid.** Network: world → country → ASN → IP, the
  paper's smoothing (μ_h = |h|), uniform interpolation, the user's world level
  rooted at the population. Device, OS and browser: flat Freeman (families,
  β = 5) as in freeman-0.3.0. `hour` leaves the score (it lowered every
  comparison; `hour_seen_before` stays for the supervised second opinion).
- Both parts blend the deployment's learned logins with the training prior
  (κ = 100, ADR-0042). The math lives once, in `rba_features.hierarchy`;
  training, the PDP and profile-service call it, and tests pin online equal to
  the offline models.
- Profiles gain per-prefix counts; `python -m rba_profile_service.rebuild`
  rebuilds existing profiles from `profile_history`.
- The console explains the network by how far the user's history reaches
  ("IP nueva en una red habitual", "país nuevo").
- The demo attackers claim a common desktop browser (Windows + Chrome), as real
  kits and Wiefling's VPN attacker do.

## Consequences

- Offline: ROC-AUC 0.807 → 0.889, recall@1%FPR 0.13 → 0.53 on the labelled
  takeovers; against Wiefling's attackers at 10% of legitimate logins challenged,
  VPN 0.42 → 0.81 and targeted 0.11 → 0.66 caught.
- Policy comparison, starter defaults: legitimate friction 35.9% → 21.4% at
  31/38 takeovers stopped (32/38 before); users with prior logins go from
  15–26% to 2–3% friction. Bands stay as ADR-0040 set them.
- A new country costs at most log 4 from the network part, from the first
  login; the score range is narrower than flat Freeman's.
- Targeted attackers on a new IP inside the victim's own ISP remain the weak
  case (the 99% tail favours flat Freeman), recorded in the finding.
