# 44. The decision path touches Valkey only; Postgres is filled behind it

- Status: Accepted
- Date: 2026-10-23
- Amends [ADR-0011](0011-async-outbox-rabbitmq.md) (the outbox becomes a Valkey
  stream) and [ADR-0018](0018-live-decision-browser-reads-pdp.md) (the PDP's
  `decisions` table stays the console's read model, written by the publisher).
- Evidence: cloud e2e load runs of 2026-10-23 (informe note 12).

## Context

Under the presenter's load on AWS (32 concurrent clients, ~100 decisions/s)
the PDP's p95 was 0.5–2.5 s against RNF-07's 200 ms. A per-step histogram
(`rba_evaluate_stage_seconds`) put the whole tail on the Postgres steps: the
replay look-up, the policy read on a cache miss, and the decision + outbox
insert. Profile, population and scoring stayed under 20 ms at p99. Pool sizing,
one transaction per decision and asynchronous commit each helped or did
nothing; the database stayed in the answer's path, shared with three
consumers on a small RDS instance. A bigger instance would only move the line.

## Decision

- **A login's answer waits on Valkey only.** Profile and population were
  already there. Policies are served from each replica's cache, refreshed by a
  background thread; the database is read on the request only for a tenant the
  replica has never seen.
- **The decision journal.** One Lua call keeps the DecisionMadeEvent under
  `rba:decision:{event_id}` (24 h) and appends it to the stream `rba:events`,
  both or neither. A replay finds the key and gets the original decision back.
  Step-up confirmation reads the decision from the journal and records
  `rba:confirmed:{id}` the same way, so it happens once.
- **The event-publisher relays the stream** as a consumer group: writes the
  batch's decisions to the `decisions` read model in one transaction
  (`ON CONFLICT DO NOTHING`), publishes each event to RabbitMQ, then
  acknowledges and deletes the entries. A failed write or publish leaves them
  pending; another pass claims them after 30 s. Consumers stay idempotent on
  `event_id`.
- **The console reads Postgres as before**, a moment behind the login. The
  decision detail falls back to the journal when the read model does not have
  the decision yet.
- The `outbox` table is no longer written (left in place on kept databases).

## Consequences

- RDS is off the login path: its size, its other writers and its connection
  setup no longer show up in the decision latency.
- **Durability moves to Valkey.** The ElastiCache node is a single node without
  persistence: if it is lost, decisions not yet relayed (normally well under a
  second's worth) are lost with it, as profiles already would be. Stated as a
  limitation; a replica with Multi-AZ or AOF persistence would close it.
- The decision list is eventually consistent (about a second). The e2e waits
  for the decision it expects instead of reading the newest row at once.
