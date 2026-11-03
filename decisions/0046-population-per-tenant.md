# 46. Each tenant has its own login population

- Status: Accepted
- Date: 2026-11-03
- Amends [ADR-0042](0042-learn-from-completed-logins-and-deployment-population.md) (one
  population for the whole deployment).

## Context

ADR-0042 made Freeman's attacker model the deployment's own learned logins,
with the training data as a Dirichlet prior (κ = 100). It kept one Redis hash,
`rba:population`, for every tenant: Roku Bank's customers moved the scores of
Kaze Labs' employees and the other way round. The hash holds aggregate counts,
not rows, but it is still learning across organizations, which the security
architect interviewed asked us not to do, and which the informe's reviewer
(C3) flagged against RF-20's isolation.

The one thing a shared population bought was a warm start: a new tenant
borrowed everyone else's counts from its first login.

## Decision

- The population is kept per tenant: `rba:population:<tenant_id>`
  (`rba_features.population.population_key`). The PDP reads and learns the
  login's tenant's hash; profile-service learns, seeds and rebuilds per tenant
  from the `tenant_id` already on every event and history row.
- Seeded home histories name their tenant (`SEED_PROFILE_USER_IDS` is
  `<tenant_id>:<user_id>,…`); an entry without one is refused.
- `python -m rba_profile_service.rebuild` deletes the old global hash and every
  tenant's, and replays `profile_history` into each tenant's.

## Consequences

- No tenant's logins move another tenant's scores. The model artifact (the
  training prior) is still one for everyone; a per-tenant model stays parked.
- **Cold start per tenant.** A new tenant scores against the training prior
  alone until it has learned logins of its own; at κ = 100 its own counts weigh
  as much as the prior after about 100 learned logins. That is the ADR-0042
  weakness (Norway-shaped prior) for each new tenant's first days, softened by
  the hosting rule and the rules that do not depend on the population.
- A kept Valkey (k3d) needs one `rebuild` after the upgrade; until then each
  tenant starts empty. The cloud stack is created fresh on every `up`.
