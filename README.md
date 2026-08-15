# RBA project documentation

Central, cross-cutting documentation for the risk-based authentication project
(org: [`roku-pfi`](https://github.com/roku-pfi)). Service-specific usage lives
in each sibling repo’s README; this repo holds the things that span services
and feed the thesis report.

**Start here:** [`plans/status.md`](plans/status.md) — the one place that says
where we are. Workspace map: [`../README.md`](../README.md).

## Contents

| Path | What it is |
|---|---|
| [`plans/status.md`](plans/status.md) | **Single source of truth** for phase/step checklist + current focus |
| [`plans/development_plan.md`](plans/development_plan.md) | Canonical roadmap (§8 = phases). Product horizons in §1.1 / ADR-0012–0015 |
| [`plans/README.md`](plans/README.md) | How to read the plan vs status |
| [`devlog.md`](devlog.md) | Chronological development log (newest on top) |
| [`decisions/`](decisions/) | Architecture Decision Records (immutable once Accepted) |
| [`findings/`](findings/) | Experiment write-ups with the numbers the thesis will cite |
| [`rules/`](rules/) | Versioned mirror of Cursor agent rules (active copies in `../.cursor/rules/`) |
| [`AGENTS.md`](AGENTS.md) | Portable orientation for AI coding tools |

## Sibling code repos

| Repo | Role |
|---|---|
| `rba-features` | Shared feature library (train/serve parity) |
| `rba-contracts` | OpenAPI / AsyncAPI / JSON Schema / Pydantic |
| `rba-decision-service` | PDP (`POST /risk/evaluate`) |
| `rba-idp` | IdP / PEP (users, apps, login, session, mock MFA) |
| `rba-event-publisher` | Outbox → RabbitMQ |
| `rba-profile-service` | Async Redis profile updates |
| `rba-audit-service` | Async audit store |
| `rba-infra` | Shared compose (Redis / Postgres / RabbitMQ) |
| `rba-ml-training` | Offline data + modelling |

## How we keep this updated

- After each **step** or work session → dated `devlog.md` entry + tick
  `plans/status.md` if a checkbox moved.
- Whenever we make a **non-trivial choice** → add an ADR (or supersede an
  existing one — never rewrite an Accepted ADR).
- After each **experiment/analysis** → add or update a `findings/` write-up
  (tables preferred; never invent numbers).

In Cursor, `/log-progress` automates this loop
(`.cursor/skills/log-progress/`).

## Thesis framing (do not drift)

- **Thesis core = RBA** (ADR-0015): explainable score + action + reasons on a
  real login path.
- **Product shell = thesis-scale IdP** (ADR-0014): like Auth0/Authentik in
  shape (users, apps, hosted login, session, MFA, admin); unlike them (no
  full OIDC/SAML/SCIM/LDAP/social).
- **IdP-3 (PDP enforce) is not skippable.** The IdP without RBA is an empty
  login app.

This repo holds no datasets or model blobs by design.
