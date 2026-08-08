# ADR-0001: Repository & deployment strategy (polyrepo + microservices + Kubernetes)

- Status: Accepted
- Date: 2026-08-05

## Context

Solo final project (PFI) with a December deadline. The architecture is expected to
evolve, and demonstrating a **scalable, orchestrated** system is itself a graded
objective — a design that only runs as a single un-scalable monolith would lose
points. The login decision path must stay fast. Two independent axes were on the
table: repository strategy (monorepo vs polyrepo) and deployment topology
(monolith vs many services on Kubernetes). These were initially conflated.

Options considered:
- Modular monolith in one repo (simplest, but weak on the scalability objective).
- Monorepo with many independently-deployable services.
- Polyrepo: one repo per service.

## Decision

- **Polyrepo**: one repo per running service, plus shared library repos
  (`rba-features`, `rba-contracts`) and an infra repo (`rba-infra`).
- **Containerized microservices deployed as pods on Kubernetes**, with the
  decision service running 2+ replicas behind an autoscaler (HPA).
- **Keep the synchronous login path lean**: feature assembly + rules + policy stay
  in-process in one `decision-service`; model inference may be a same-pod
  **sidecar**. "Many services + fast login" is achieved via horizontal scaling and
  async off-loading, NOT by splitting the hot path into network hops.
- Everything off the request path (audit, profile updates, analytics, dataset
  building, training) is its own service, free to be fine-grained.

## Consequences

- Positive: real microservices + k8s story for grading; independent deploys;
  database-per-service; clean evolution path.
- Negative (the "polyrepo tax"): N× CI/CD, contract drift risk, train/serve feature
  skew risk, harder local dev. Mitigations: `rba-features` (versioned shared feature
  package + parity tests), `rba-contracts` (schemas + contract tests), `rba-infra`
  (reusable CI template + GitOps + Tilt/Skaffold local dev).
- Do-not-cut items: k8s + HPA, `rba-features` parity, the leakage experiment.
