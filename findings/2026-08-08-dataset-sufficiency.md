# Is the Wiefling dataset sufficient? (report note)

- Date: 2026-08-08
- Purpose: record the reasoning behind using the Wiefling RBA dataset despite the
  scarcity of account-takeover labels, so the thesis can state precisely what the data
  can and cannot support.
- Related: ADR-0003 (dataset selection), ADR-0004 (modelling/label strategy),
  ADR-0007 (evaluation protocol), findings `2026-08-08-phase1-eda.md` and
  `2026-08-08-step5-baselines.md`.

## Two separate questions

Dataset adequacy conflates two things that have different answers:

- **Validity** — is it a legitimate, faithful dataset? **Yes.** It is the standard
  peer-reviewed RBA benchmark, a privacy-preserving synthesis that preserves the real
  statistical structure and per-user relationships of a 3.3M-user / ~33M-login SSO, and
  the authors show it reproduces results obtained on the original real data. Our own
  feature validation confirms the signal is real (`ip_seen_before` 0.08 for takeovers
  vs 0.45 for legitimate logins).
- **Sufficiency** — is it enough for what we claim? **Partly, and claim-dependent.** The
  binding constraint is the number of positive labels.

## The binding constraint: 141 positives is a global ceiling

There are **141 account-takeover rows in the entire ~33M-login dataset** (138 users),
and the modelling subset already preserves **all** of them. Consequences:

- **Resampling cannot add positives.** Scaling from 50k users to the full 4.3M adds
  legitimate logins and `is_attack_ip` rows, but the takeover ceiling stays at 141.
  The scarcity is structural, not a sampling artifact.
- Labels also only span **2020-02-04 → 2020-11-23** (none afterwards), which forces the
  label-covered evaluation window (ADR-0007).
- **This scarcity is realistic.** Real account takeover is extraordinarily rare relative
  to normal logins; an honest real-world dataset looks like this. A dataset with a large
  positive fraction would be the unrealistic one.

## What the thesis actually needs from the data

The project goal is an **explainable, scalable, adaptive RBA system**, not
state-of-the-art takeover detection. Against that goal the dataset is sufficient:

1. **Prove the mechanism is real** — done (seen-before features separate attacks; a
   trivial LogReg reaches 50% recall @ 1% FPR — see Step 5 finding).
2. **Ground the system in realistic data** — done (realistic distributions, imbalance,
   per-user behaviour, temporal patterns).
3. **Drive the engineering** — latency, model size, feature parity, policy/challenge
   flow, and the microservice/k8s architecture. The data is more than enough for these.

The 141 limits exactly **one** thing: precise, tuned detection metrics with tight
confidence intervals. That is handled by design (below), not hidden.

## Mitigations (already in the plan)

- **Freeman as primary scorer** — label-free by construction, so the positive count
  barely constrains it (ADR-0004). The scarcity makes this choice stronger, not weaker.
- **`is_attack_ip` as a populous auxiliary signal** — 17,570 rows across the full year;
  a different, noisier target (attack-IP ≠ takeover), used for the leakage A/B (Step 6)
  and for exercising the pipeline at scale.
- **Own synthetic scenario generator** (later phase) — inject clearly-labelled attack
  scenarios (credential stuffing, impossible travel, device swap) *on top of* the real
  backbone to stress-test the system end-to-end and probe recall by attack type. Framed
  as a complement, not a replacement — standard practice for rare-label security/fraud
  problems.
- **Report uncertainty explicitly** — bootstrap confidence intervals on Step 5 metrics
  so small-n noise is visible, turning the honesty into a rigor point.

## Bottom line for the report

The Wiefling dataset is the correct choice with no meaningfully better public
alternative. State plainly: it is excellent for demonstrating the RBA mechanism and for
driving the system engineering, and inherently thin for tuned detection numbers — a gap
addressed with a label-free primary model, honest uncertainty reporting, and a
complementary synthetic generator. This is a stronger, more defensible position than
assuming abundant labels.
