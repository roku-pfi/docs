# ADR-0020: Local Kubernetes via k3d + Helm in `rba-infra`

- Status: Accepted
- Date: 2026-08-16

## Context

ADR-0001 requires containerized microservices on Kubernetes (decision-service
at 2+ replicas behind an HPA). ADR-0010 put the shared data plane in
`rba-infra` as Docker Compose and deferred k3d / Tilt / Helm until the IdP
shell existed. IdP-1…7 are done; the leftover graded story is a real local
cluster, not more IdP features.

Options considered:

- **kind** — common, extra ingress/storage install.
- **k3d** (k3s-in-Docker) — Traefik, local-path provisioner, and metrics-server
  come with k3s; fits OrbStack / Docker on an M2 laptop.
- **Tilt / Skaffold now** — better inner loop, another tool to install before
  any pod exists.
- **Bitnami (or similar) dependency charts** for Postgres/Redis/RabbitMQ —
  faster to copy, heavier, and a second source of truth vs compose images.

## Decision

1. **k3d** single-node cluster `rba`, host port **8080 → Traefik 80**.
2. **One Helm chart** `rba-infra/helm/rba`: same Redis / Postgres / RabbitMQ
   images as compose, plus Deployments for every running service. Dockerfiles
   live in the **service repos**; charts stay in `rba-infra`.
3. **Ingress only for `rba-idp`.** The PDP, workers, and datastores are
   ClusterIP. The IdP reaches the PDP at `http://decision-service:8000`.
4. **Compose remains** the no-cluster inner loop (venv + `docker compose up`).
5. **Defer** Tilt, Prometheus/Grafana, GitOps, NetworkPolicies, and a shared
   policy store. `PUT /policy` stays in-process + per-pod YAML (default image
   policy is identical across the 2 PDP replicas).

Bring-up: `rba-infra/scripts/k3d-up.sh` (needs Docker/OrbStack, k3d, helm,
kubectl). Teardown: `scripts/k3d-down.sh`.

## Consequences

- Positive: the architecture claim in ADR-0001 is now a runnable local
  cluster; HPA object exists on `decision-service` (min 2 / max 5, CPU 70%);
  probes and resource requests/limits are on the HTTP services.
- Negative: another local mode besides compose; workers have no HTTP probes
  yet; HPA does nothing interesting until Phase 5 load tests; live policy PUT
  does not fan out across PDP replicas.
- Follow-up: Prometheus / Grafana + a load test that actually moves the HPA
  (Phase 5); Tilt; GitOps/CI templates; DLQ.
