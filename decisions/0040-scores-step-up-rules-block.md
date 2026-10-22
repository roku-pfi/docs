# 40. A risk score steps up; only rules block

- Status: Accepted
- Date: 2026-10-22
- Amends: [ADR-0034](0034-base-policy-and-engine-mode.md) (default bands and
  level→action of the seeded and starter policies).
- Evidence: [finding 2026-10-22 — policy comparison](../findings/2026-10-22-policy-comparison.md) (D-2).

## Context

The default policies mapped the top score band to BLOCK. Replaying the held-out
logins through the production decision code showed that this band blocked **6.21 %**
of legitimate logins and stopped **no** takeover that a step-up would not have: with
CRITICAL → REAUTHENTICATE the engine stops the same 32 of 38 takeovers and blocks
**0.02 %** (the failed-login lockout rule). The same replay showed the LOW band's
upper limit can rise from 0.30 to 0.40 with no loss of detection (friction 38.1 →
36.0 %), and that above 0.40 detection falls off a cliff.

## Decision

- A score never blocks: CRITICAL maps to REAUTHENTICATE (passkey) in every seeded
  tenant policy and in the starter policy for new tenants.
- Blocks come from rules whose evidence is direct (the failed-login lockout).
- Default bands: LOW ≤ 0.40, MEDIUM ≤ 0.60, HIGH ≤ 0.80, CRITICAL above.
- Seeded policies move to version 3.0.0. A tenant can still choose BLOCK for
  CRITICAL in Política; the console's Efectividad page shows what that costs.

## Consequences

- Legitimate users are asked for their passkey instead of being locked out when the
  score is very high; an attacker without the passkey is stopped either way.
- The demo's four outcomes still all occur: the BLOCK comes from the lockout
  scenario, as in the scenario map (ADR-0035).
- Kept databases (k3d) keep their current policy; the new defaults apply to fresh
  ones (every cloud session) and new tenants.
