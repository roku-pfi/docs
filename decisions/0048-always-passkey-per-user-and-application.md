# 48. "Passkey always" per user and per application

- Status: Accepted
- Date: 2026-11-03
- Related: RF-08, [ADR-0034](0034-base-policy-and-engine-mode.md) (base policy
  per tenant), [ADR-0037](0037-passkey-enrollment-from-trusted-context.md).

## Context

"Passkey siempre" existed only as a tenant's base policy. RF-08 asks for it per
user and per application too: an administrator or a sensitive application
should always step up, whatever the risk.

## Decision

- `always_passkey` on users and applications in the IdP directory, set from the
  console (Usuarios and Aplicaciones: "Pedir passkey siempre"), exposed in the
  admin API (contracts 0.21.0).
- **The IdP enforces it** as the PEP: an ALLOW becomes REQUIRE_MFA; anything
  stricter stays. It holds with the engine off, down or in monitor mode.
- **The PDP records it.** The IdP sends `step_up_required`; the PDP floors the
  *enforced* action and adds the reason `always_passkey`. Its *verdict* stays as
  the engine judged the login, so Eventos still shows what the engine thought,
  the challenge is not labelled "Inicio poco habitual", and the login is
  learned only once the passkey is passed (ADR-0042).

## Consequences

- RF-08 is met at every level: tenant, application, user.
- A user without a passkey on an always-passkey user or application gets the
  "no second factor" screen, as under a tenant's ALWAYS_MFA: they need an
  activation code.
