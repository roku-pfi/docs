# ADR-0010: Shared local data plane lives in `rba-infra`

- Status: Accepted
- Date: 2026-08-11

## Context

Phase 3 temporarily shipped `docker-compose.yml` inside `rba-decision-service`
(Redis + Postgres) so `/risk/evaluate` could run without a cluster. That contradicts
the polyrepo plan: platform concerns belong in `rba-infra`
(`development_plan.md` §4.1, §9), and Redis is a **shared** online-profile cache
(written later by profile-service), not a decision-service private dependency.
Database-per-service means separate *databases*, not a Postgres container per app
repo.

## Decision

1. Bootstrap **`rba-infra`** with a shared local compose stack: one Redis, one
   Postgres server, init script creating `rba_decision` / `rba_profile` /
   `rba_audit` databases.
2. **Remove** compose from `rba-decision-service`. The service only consumes
   `REDIS_URL` / `DATABASE_URL`.
3. Defer the rest of Phase 0 (k3d/Tilt/Helm/GitOps/CI templates) until needed for
   multi-service deploys; compose is the light foundation now.

## Consequences

- Positive: correct ownership; future services reuse the same local stack; ports
  no longer fight across repos.
- Negative: developers must `docker compose up` from `rba-infra` before running a
  service (documented in both READMEs).
- Follow-up: add RabbitMQ to the same compose in Phase 4; grow Helm/Tilt here for
  the graded k8s story.
