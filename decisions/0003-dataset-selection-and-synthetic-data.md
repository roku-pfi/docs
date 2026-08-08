# ADR-0003: Dataset selection & use of synthesized data

- Status: Accepted
- Date: 2026-08-08

## Context

We need labeled login data with attack/takeover signals. Real production login
datasets are essentially unreleasable for privacy reasons. The Wiefling et al.
"Login Data Set for Risk-Based Authentication" (Zenodo 6782156) is the standard open
benchmark, but its Zenodo page warns that feature values are "plausible but totally
artificial," which raised the concern of training on synthetic data.

## Decision

Use the **Wiefling RBA dataset** as the backbone. It is **not** rule-invented
synthetic data: it is a **privacy-preserving synthesis of real behavior** (3.3M
users / ~33M logins at a Norwegian SSO) that **preserves the statistical properties
and per-user relationships** of the original, and the authors show it reproduces the
results obtained on the real data. Training on it is legitimate and standard.

## Consequences

- Faithful (rely on): per-user value consistency → all `*_seen_before` features;
  class imbalance; temporal patterns; attack/takeover labels.
- Degraded (do not over-trust): real IP reputation; true geolocation; absolute geo
  distance ("impossible travel"); exact timestamps/RTT positions (regenerated).
- Report requirements: describe it accurately as "a privacy-preserving synthesized
  dataset that preserves the statistical properties of a real 33M-login service";
  cite Wiefling et al. (2022); do not claim real-world geo/IP-reputation performance.
- This is documented for contributors in `rba-ml-training/data/README_personal.md`.
- Our own scenario generator (later) will *complement*, not replace, this backbone.
