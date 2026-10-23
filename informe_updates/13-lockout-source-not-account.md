# Failed logins: block the source, not the account (ADR-0045, 2026-10-23)

- **Before:** failures counted per account from anywhere; 10 in a day blocked
  the account. An attacker could lock any customer out by guessing (denial of
  service), which the demo's sustained scene showed with Martín.
- **Now:** per source IP across users (10 in 15 min → that IP blocked 30 min,
  right password included); failures from contexts the user never used → their
  next login needs the passkey, never a block; only the owner's own failures
  from a familiar network (5 in 15 min) pause the account for 15 min. All
  per-tenant policy values.
- **RF-07 (retardo progresivo) se cubre así**, sin retardo por intento: un
  retardo sostenido retiene un hilo del servidor por intento (palanca de DoS)
  y frenaría al atacante antes de que su IP llegue al umbral de bloqueo.
- **Review:** the decision-policy chapter (the failed-login rules and their
  table of actions), RF/threat sections on credential stuffing and lockout
  abuse (cite OWASP / NIST SP 800-63B throttling), the demo script (sustained
  scene: attacker IP blocked, Martín passes with his passkey), limitations
  (shared IPs / carrier NAT; IP rotation below the threshold is left to the
  passkey).
