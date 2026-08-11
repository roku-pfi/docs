# ADR-0011: Phase 4 async plane — outbox → RabbitMQ → profile/audit

- Status: Accepted
- Date: 2026-08-11

## Context

Phase 3 wrote decision + outbox in one Postgres transaction and optionally
updated Redis synchronously (`PROFILE_WRITE_MODE=sync`). The architecture
requires async fan-out so the login path stays independent of broker/consumer
uptime (`development_plan.md` §2–3, §8 Phase 4). Profile writes ultimately belong
to `profile-service`; audit is a separate consumer. The DecisionMade contract also
lacked the raw login signals needed to call `update_profile`.

## Decision

1. Add **RabbitMQ** to `rba-infra` compose (shared broker).
2. Ship **`rba-event-publisher`**: poll unpublished outbox rows in
   `rba_decision`, publish to topic exchange `rba.events` with routing key
   `rba.decision.made.v1`, set `published_at`.
3. Ship **`rba-profile-service`**: consume that routing key, idempotent on
   `event_id`, apply `rba_features.update_profile` to Redis, append history in
   `rba_profile`.
4. Ship **`rba-audit-service`**: consume same events into `rba_audit`.
5. Extend **`DecisionMadeEvent`** with optional `login: LoginEventSnapshot`
   (`rba-contracts` 0.1.1); decision-service always populates it. Profile
   consumer requires it to update state.
6. Production-like local path uses `PROFILE_WRITE_MODE=none` on decision-service
   so Redis is owned by profile-service.

## Consequences

- Positive: request path stays thin; broker downtime only delays fan-out; correct
  ownership of Redis writes.
- Negative: eventual consistency window between decision response and profile
  update (acceptable for thesis; document for demos).
- Follow-up: dead-letter queues, metrics, k8s Deployments for the three workers.
