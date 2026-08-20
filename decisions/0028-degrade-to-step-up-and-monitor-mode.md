# 28. Degrade to a step-up, and add monitor-only mode

- Status: Accepted
- Date: 2026-08-20
- Supersedes: the "fail closed → HTTP 503" invariant stated in the workspace
  README and in `rba-idp/AGENTS.md` (never an ADR, but a documented invariant).
- Related: [ADR-0009](0009-online-profile-freeman-serving.md) (PDP-internal
  fallback), [ADR-0027](0027-supervised-second-opinion-and-failed-logins.md)
  (escalation ladder), [ADR-0023](0023-end-user-login-is-opaque.md).
- Closes: **RF-09, RF-10, RNF-03, RNF-08** (E50 §3.2).

## Context

Two requirements came out of the interviews and neither was implemented. Both
are about the same thing — what the system does when it is *not* confident — so
they are decided together.

**RF-10 / RNF-03.** «El sistema debe exigir un MFA equivalente si el motor de
riesgo o de políticas no responde, sin permitir el acceso abierto ni un bloqueo
masivo de usuarios legítimos por esa indisponibilidad.»

We had half of this. When the PDP's *own* dependencies fail (Redis, the scorer),
`apply_policy(..., fallback=True)` already degrades to the bundle's
`fallback_action`, which every shipped bundle sets to `REQUIRE_MFA`. But when the
PDP **process** is unreachable, `rba-idp` raised HTTP 503. That is precisely the
`bloqueo masivo` the requirement forbids: a risk *engine* outage becomes an
*authentication* outage, and every legitimate user is locked out for its
duration. The workspace README even recorded this as a deliberate invariant
("Fail closed"), which read well and was wrong — "fail closed" is the right
instinct for an authorization decision, but a login is not a resource, and the
password has already been verified by the time the PDP is consulted.

**RF-09 / RNF-08.** «El sistema debe ofrecer un modo de solo monitoreo que
registre la decisión del motor sin ejecutarla sobre el usuario.» Nothing like it
existed. This is the requirement that makes the ML layer deployable: an operator
will not point a model at real users on day one, and RNF-08 explicitly names
monitor mode as the way to "observar el modelo antes de aplicarlo".

## Decision

### 1. The PEP degrades to a step-up, never to 503 or ALLOW

`rba-idp` catches `PdpUnavailable` on the login path and synthesises a decision
locally: `action = REQUIRE_MFA`, `fallback=true`, `risk_score=0.0`, and one
reason `pdp_unavailable`. The user completes the same passkey ceremony they
would for any other step-up.

The action is configurable via `PDP_UNAVAILABLE_ACTION`, typed
`Literal["REQUIRE_MFA", "REAUTHENTICATE"]`. **`ALLOW` and `BLOCK` are not
expressible.** The requirement forbids both outcomes, so we let the type system
forbid them rather than trusting an operator not to configure them — a
misconfiguration cannot open the door or close it on everyone.

Two boundaries stay as they were:

- **Admin read proxies still return 503.** A dashboard that cannot load is not a
  login that cannot happen. Failing those loudly is correct.
- **A wrong password is still `INVALID_CREDENTIALS`.** Degrading applies only
  after the credential check passes; an outage must not become a bypass.

The synthesised decision is **not** written to the decision store — the store
lives behind the service that just failed to answer. It survives in the IdP log
and on the MFA challenge row. This is a real gap in RF-11 coverage during an
outage and is accepted rather than solved with a second write path.

### 2. Monitor-only mode is policy state, not a deploy flag

`PolicyBundle.monitor_only` (default `false`), overridable per application. When
on, the PDP does everything it normally does — features, Freeman, the supervised
opinion, the rules, the escalation ladder, persistence, the outbox — and then
returns `ALLOW` to the PEP with the engine's verdict in a new
`RiskEvaluateResponse.monitored_action` field and a `monitor_only` reason at the
head of the reason list.

Three consequences worth stating explicitly:

- **The record holds the verdict, not the ALLOW.** `DecisionRow.action` and the
  published event carry what the engine decided, because RF-09 says the point is
  to *record the engine's decision*. The `monitor_only` reason is the marker that
  says it was not enforced, and `_row_to_response` reads it back on idempotent
  replay so a replayed decision allows exactly like the live one did.
- **Metrics count the verdict too**, with a new `enforced` label on
  `rba_decisions_total`. A monitor-mode rollout should show the true shape of
  what the policy *would* do — that is the entire point of running it.
- **It lives in the policy bundle**, so `PUT /policy` toggles it with no
  redeploy (RNF-05) and a pilot can be scoped to one tenant.

### 3. Monitor mode also suppresses the fallback action

If the scorer fails while a tenant is in monitor mode, the PDP returns `ALLOW`,
not `fallback_action`. Monitor mode is an explicit "do not act on my users yet";
a Redis outage is not a reason to start acting. Waking up a monitored pilot into
live challenges is the exact surprise the mode exists to prevent.

This does **not** weaken RNF-03, because RNF-03's guarantee is kept where it
means something: at the PEP, when the PDP is silent. There, monitor mode is
unknowable — policy state lives in the service that is not answering — so the
IdP applies the safe default and asks for a factor.

## Consequences

**Good.**

- RF-10 / RNF-03 satisfied end to end, and the guarantee is encoded in a type
  rather than in a comment.
- RF-09 satisfied; RNF-08's last open clause closes with it.
- Demonstrable live: stop the PDP mid-walkthrough and the login still completes,
  through a passkey. Flip `monitor_only` and the stuffing lockout scenario keeps
  producing `BLOCK` in the admin decision browser while the user sails through.
- `rba-contracts` changes are additive: `monitor_only` defaults to `false` and
  `monitored_action` defaults to `None`, so an un-upgraded consumer sees exactly
  the old behaviour.

**Costs and risks.**

- A PDP outage now degrades silently from the user's point of view — they see a
  step-up, not an error. Operators need the log line and the `pdp_unavailable`
  reason to tell the two apart. This is the intended trade (RNF-01 explainability
  is toward the operator, ADR-0023) but it does mean an outage is less visible
  than a 503 was.
- An outage window leaves a hole in the decision trail (RF-11).
- Monitor mode is a foot-gun by construction: a bundle left with
  `monitor_only: true` enforces nothing. Mitigated by defaulting to `false`, by
  the `enforced` metric label, and by a test asserting the shipped policy
  enforces.
- Adding the `enforced` label creates new timeseries. The existing Grafana panel
  aggregates `sum by (action)`, so it is unaffected.

## Alternatives rejected

- **Keep the 503 and document it as a limitation.** It contradicts a written
  requirement sourced from an interview, and the fix is small.
- **Degrade to `ALLOW` with a warning.** Forbidden by RNF-03, and it converts an
  availability incident into a security incident.
- **Monitor mode as an env var on the PDP.** Would need a redeploy to toggle
  (violating RNF-05) and could not be scoped to one application.
- **A separate `monitored` column on `DecisionRow`.** The service uses
  `create_all` with no migration tooling; a new column would not appear on
  existing databases. Encoding it as a reason needs no schema change and shows up
  in the admin decision browser for free.
