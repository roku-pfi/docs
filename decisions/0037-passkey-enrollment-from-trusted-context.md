# 37. A passkey is enrolled only from a trusted context

- Status: Accepted
- Date: 2026-10-21
- Amends: [ADR-0022](0022-product-demo-over-gitops.md) / Demo-4 (passkey offered
  at the first step-up).
- Related: [ADR-0034](0034-base-policy-and-engine-mode.md),
  [ADR-0033](0033-multi-tenancy.md).
- Requirements: **RF-06**, **RNF-03**.

## Context

Since Demo-4, a user who has no passkey and is asked for a step-up is offered to
**create** one on the spot (`rba_idp/services/login.py`, the `CREATE` branch of
the WebAuthn options endpoint). That is trust on first use at the worst possible
moment: the step-up exists because the login looks suspicious, and the person
being offered enrollment is whoever typed the password.

An attacker holding Martín's password who triggers the first step-up registers
*their* authenticator, and from then on passes every step-up as Martín. In the
cloud deployment the database is recreated each session, so every account starts
in that state.

## Decision

A passkey can be registered only from one of two trusted contexts:

1. **After a low-risk login.** When the engine's verdict is `ALLOW` (usual
   context) and the user has no passkey, the IdP offers "Agregá una passkey"
   after the session is established. Never inside a step-up.
2. **With an activation code** issued by the tenant's operator from the console:
   single use, short-lived (15 minutes), stored only as a hash, bound to one
   user. Covers `ALWAYS_MFA` tenants, where no login without a second factor
   exists to enroll from, and lost devices.

**A step-up demanded of a user with no passkey is denied**, with a generic
"no pudimos verificar tu identidad, contactá a soporte" and no session. It is
never turned into an enrollment. The decision record keeps the engine's verdict;
the denial reason (`no_second_factor`) goes to the IdP log and the MFA challenge
row.

## Consequences

- The enrollment hole closes: possessing the password is no longer enough to
  bind a new authenticator.
- `ALWAYS_MFA` tenants need an operator step before a user's first login. In
  the demo this is a scene of its own: Diego issues Martín's activation code.
- Each cloud session starts with enrollment (one activation code per demo user,
  or a usual-context login for `PASSWORD_ONLY` tenants).
- Adds one console action, one IdP page and one table (`activation_codes`).
