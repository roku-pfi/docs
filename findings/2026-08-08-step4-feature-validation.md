# Step 4 — feature validation on real data

- Date: 2026-08-08
- Source: `rba-features` replay driver run over `rba-ml-training` subset
  (202,284 rows, built in ~27s via per-user `replay_user`).

## What we checked

Whether the Phase 1 features (per-user "seen-before" signals + behavioural ones)
actually separate malicious from normal logins on the real subset. This both
validates the feature implementation and confirms the modelling direction (ADR-0004).

## Mean feature value by class

### by `is_account_takeover` (the honest label)

| feature | legit (False) | takeover (True) |
|---|---|---|
| ip_seen_before | 0.452 | **0.078** |
| asn_seen_before | 0.682 | **0.135** |
| country_seen_before | 0.747 | **0.177** |
| browser_seen_before | 0.501 | **0.213** |
| hour_seen_before | 0.397 | **0.227** |
| os_seen_before | 0.608 | 0.411 |
| device_type_seen_before | 0.701 | 0.504 |
| user_login_count | 10.68 | 6.79 |
| failed_logins_last_24h | 0.277 | 0.184 |

### by `is_attack_ip` (noisier proxy)

Same direction but much weaker separation (e.g. country_seen_before 0.760 vs 0.613,
ip_seen_before 0.463 vs 0.334) — consistent with attack-IP being a noisier signal
than true takeover.

## Takeaway

- The RBA hypothesis holds on this data: **account takeovers overwhelmingly occur
  from contexts (IP/ASN/country/browser/hour) the user has never been seen in.**
- The `*_seen_before` family are strong, explainable discriminators → ideal inputs
  for the Freeman likelihood-ratio scorer.
- Performance note: naive per-user Python replay does 202k rows in ~27s; fine
  offline, worth optimizing later if the subset grows.
