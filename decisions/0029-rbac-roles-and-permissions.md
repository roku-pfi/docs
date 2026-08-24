# 29. RBAC: roles and permissions, not groups pretending to be roles

- Status: Accepted
- Date: 2026-08-20
- Supersedes the app-scoped `access` grant introduced in
  [ADR-0019](0019-groups-grant-app-access.md).
- Closes: **RF-13, RF-21** (E50 §3.2) and makes §1.2's "control de acceso basado
  en roles" literally true.

## Context

The scope document promises *control de acceso basado en roles*; RF-13 promises
lifecycle management of "usuarios, grupos, **roles y permisos**"; RF-21 promises
authorization "a recursos y operaciones según los roles y permisos del usuario
autenticado".

What existed was one table, `group_app_grants`, whose `permission` column was
pinned to the constant string `"access"`. There was no role, no resource and no
operation anywhere in the codebase. Authorization had exactly one granularity —
*may this user open this application at all* — and platform administration was a
boolean column, `users.is_admin`.

That is a defensible thesis-scale simplification, but it is not RBAC, and a
tribunal that knows the difference will ask. Three specific consequences:

- **RF-21 was unsatisfiable.** Nothing anywhere authorized an *operation*.
- **The support operator could not exist.** §3.2.1 defines four actors, one of
  which reads a decision to explain an incident (HU-10). With a boolean flag,
  that person is either a full administrator or locked out of the console.
- **`is_admin` was a second, parallel authorization system** — groups for tenant
  apps, a flag for the console. Two mechanisms is one too many.

The alternative considered and rejected was renaming groups to roles. It is a
one-line change and it makes the document read correctly, but groups (*who*) and
roles (*what you may do*) are genuinely different things; conflating them leaves
RF-21 unsatisfied anyway and makes the design harder to defend, not easier.

## Decision

### The chain

```
user ──belongs to──▶ group ──holds──▶ role ──carries──▶ permission
                                      (scoped to one application)
```

This is NIST RBAC with user groups. Groups answer *who*, roles answer *what you
may do*, and the two stay separable: adding someone to a team and deciding what
that team may do are different operations performed by different people at
different times.

Four new tables in `rba-idp`: `permissions`, `roles`, `role_permissions`,
`group_roles`. `group_app_grants` is removed.

### Roles are application-scoped; permissions are `resource:operation`

A `Permission` is `(application_id, resource, operation)` — the wire form a
client checks is the derived string `"transfer:create"`. A `Role` belongs to
exactly one application, and `set_role_permissions` **rejects** a permission from
a different application with HTTP 422. `viewer` on the bank and `viewer` on the
console are different roles that may safely share a name; role names are unique
per application, not globally.

Without that scoping, one tenant's administrator could mint a role carrying
another application's permissions. It is the failure mode that makes an RBAC
implementation worthless, so it is enforced in the service and covered by a test.

### Holding a role *is* the sign-in grant

There is no `app:access` permission. `has_application_access` asks whether any of
the user's groups holds any role on the application. A role with an empty
permission set is a valid "may sign in, may do nothing else" assignment.

This removes a whole class of bug: nobody can be given a role and still be unable
to log in because a separate access row was forgotten.

### Permissions reach relying parties flattened, and fresh

`GET /session` now returns `application_id` and `permissions` — the union of
every permission from every role from every group, as scope strings, **scoped to
the session's own application**. A bank session never exposes console
permissions, whatever roles the user holds elsewhere.

Resolution happens inside the same transaction as the session lookup, not baked
into the token at login. A role revoked a moment ago stops working on the next
request; there is no window in which a stale token still carries the old rights.

`rba-demo-banking` consumes exactly this and nothing else: it never sees a role
or a group. It gates the balances on `account:read` and the transfer on
`transfer:create` — **on the POST handler, not only on whether the button was
drawn.** Hiding an affordance is presentation; the check on the write path is the
control.

### The console is gated by permissions too

`is_admin` is no longer an authorization input. Three guards replace it:

| Guard | Permission | Who has it |
|---|---|---|
| `require_admin` | `admin:read` | platform-admin, support-operator |
| `require_write` | `admin:write` | platform-admin |
| `require_policy_write` | `policy:write` | platform-admin |

Splitting `policy:write` from `admin:write` is deliberate: changing *who may log
in* and changing *how risky a login must be before it is challenged* are
different powers, and RNF-05 makes the second editable at runtime.

`users.is_admin` survives as a directory label. Setting it through the admin API
adds or removes membership of the platform-admins group, so the label can never
disagree with what the user can actually do — ticking the box really does confer
the power, and clearing it really does remove it.

### The seed is the worked example

| Actor (§3.2.1) | User | Group | Role | Permissions |
|---|---|---|---|---|
| usuario final | `demo@example.com` | Banking customers | `customer` | `account:read`, `transaction:read`, `transfer:create` |
| usuario final | `viewer@example.com` | Banking viewers | `viewer` | `account:read`, `transaction:read` |
| operador de soporte | `support@example.com` | Support operators | `support-operator` | `admin:read`, `decisions:read` |
| administrador | `admin@example.com` | Platform admins | `platform-admin` (+ `customer`) | all |

The two bank users are the demonstration. Identical login, identical risk
handling, and one of them gets a 403 on a transfer — RF-21 visible in ten
seconds.

## Consequences

**Good.**

- RF-13 and RF-21 are satisfied and demonstrable, and §1.2's RBAC claim is true.
- The support operator is a real principal, so HU-10's actor can be shown rather
  than described.
- One authorization mechanism instead of two.
- The admin console gains a Roles view; write controls disable themselves for
  principals who lack `admin:write`, so the UI tells the truth about permissions.

**Costs and risks.**

- **Breaking contract change** (`rba-contracts` 0.8.0): `GroupDetail.grants` →
  `roles`, `AppPermission` / `GroupGrantPublic` / `CreateGroupGrantRequest`
  removed, `/admin/api/groups/{id}/grants` → `/roles`.
- **Existing databases need re-seeding.** `create_all` adds the four new tables
  but cannot populate them, so an already-running `rba_idp` has no roles and
  every login returns `ACCESS_DENIED`. Drop and recreate the database, or insert
  the seed rows by hand. `group_app_grants` is left in place, unused.
- One extra join on the session path. It is a five-table join on primary keys
  against a thesis-scale directory; if it ever mattered, the place to cache is
  the session row.
- More surface to administer. Mitigated by seeding a complete, working example
  rather than an empty permission catalogue.

## Alternatives rejected

- **Rename groups to roles.** Cheapest, but conflates two concepts and leaves
  RF-21 unsatisfied. It answers the wording complaint without answering the
  requirement.
- **Assign roles directly to users** as well as through groups. Two paths to the
  same grant means two places to look when revoking one. Groups are the only
  path.
- **Keep `is_admin` as a bypass** alongside RBAC. A second authorization
  mechanism that silently outranks the first is exactly what this ADR removes.
- **Wildcard or hierarchical permissions** (`admin:*`, role inheritance). More
  expressive, materially harder to reason about, and nothing in the requirements
  asks for it.
