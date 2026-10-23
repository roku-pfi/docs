# Passkey enrollment from a trusted context (ADR-0037)

- **Changed:** a passkey is registered only after a low-risk login or with a
  single-use 15-minute activation code from the console. A step-up for a user
  without a passkey is denied.
- **Review:** MFA section next to the RF-06 rewrite in
  `e50-document-corrections.md`; account-recovery / lost-device text.
