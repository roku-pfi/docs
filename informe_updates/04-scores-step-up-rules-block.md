# A score steps up, only rules block (ADR-0040)

- **Changed:** CRITICAL maps to REAUTHENTICATE, never BLOCK. Blocks come only
  from direct-evidence rules (failed-login lockout: 3 failures → reauth burst,
  10 → block). Bands: LOW ≤ 0.40, MEDIUM ≤ 0.60, HIGH ≤ 0.80. Policies v3.0.0.
- **Review:** the decision-policy table/figure and any example where a high
  score blocks; RF-07 wording (progressive delay is still not built, parked).
