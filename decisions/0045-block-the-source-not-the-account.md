# 45. Repeated wrong passwords: block the source, not the account

- Status: Accepted
- Date: 2026-10-23
- Supersedes the failed-login bands of [ADR-0027](0027-supervised-second-opinion-and-failed-logins.md)
  (burst → REAUTHENTICATE, lockout → BLOCK, counted per account).

## Context

Failures were counted per account wherever they came from: three in a day
stepped the next login up, ten blocked it. The "sustained stuffing" scene
showed the consequence: twelve wrong passwords from attacker IPs abroad, and
Martín's own login from his usual laptop was blocked. Anyone who knows an
address can lock its owner out: a denial of service the engine hands the
attacker. OWASP's guidance and NIST SP 800-63B both point to throttling the
source instead of locking the account.

## Decision

Three counters in Valkey (sliding windows, per tenant), kept by the PDP on
the decision path (ADR-0044) so a block applies to the very next attempt:

- **Per source IP**, across the tenant's users. Past `ip_failures` (10) in
  `ip_window_seconds` (15 min) the IP is blocked for `ip_block_seconds`
  (30 min): every attempt from it is refused, **the right password included**.
  Reason `source_blocked`.
- **Per user, from contexts they have never used** (neither the IP nor the
  network is in their profile). These are an attack on the user, not their
  mistake: past `foreign_failures_step_up` (3) in a day their next login must
  pass the passkey (`REAUTHENTICATE`), **never a block**. Reason
  `failed_login_burst` (same code; now counted only from unfamiliar contexts).
- **Per user, from a network they do use**: the owner's own wrong passwords,
  or someone at their machine. Past `owner_failures` (5) in
  `owner_window_seconds` (15 min) the account pauses for
  `owner_cooldown_seconds` (15 min). Reason `account_cooldown`. A timed
  cooldown, like a phone's PIN delay, not a block that needs support.

The thresholds are `TenantPolicy.lockout` (contracts 0.16.0), editable per
tenant like the rest of the policy. The event id is the counter member, so a
replayed failure counts once. Blocks are engine verdicts: under MONITOR they
are recorded, not enforced (ADR-0034). The presenter's reset clears them.

## Consequences

- An attacker can no longer lock a customer out by guessing; what gets stopped
  is the attacker's IP. The demo's sustained scene now ends with each attacker
  IP refused even with Martín's real password, and Martín still getting in with
  his passkey.
- An attacker rotating IPs below 10 failures each is not blocked by the IP
  counter; the account counter still forces the passkey, which the attacker
  does not have.
- Shared IPs (carrier NAT, an office) can be blocked for everyone behind them
  for 30 min by one attacker among them; the thresholds are per tenant so a
  tenant can relax them.
- Familiar is judged on the network only (IP or ASN in the profile); a thief
  on the owner's home Wi-Fi counts as the owner, and pauses the account.
