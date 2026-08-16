# ADR-0021: Prometheus + Grafana in the `rba` Helm chart

- Status: Accepted
- Date: 2026-08-16

## Context

ADR-0020 stood up a local k3d cluster and an HPA on `decision-service`, and
deferred Prometheus / Grafana. Phase 5 / K8s-2 needs scrape, dashboards, and a
load test that actually moves that HPA. Options:

- **kube-prometheus-stack** — operator CRDs, Alertmanager, node-exporter; too
  heavy for a single-node laptop cluster and a second source of truth vs the
  in-chart data plane.
- **prometheus-community / grafana Helm dependencies** — less than the stack,
  still a second packaging style next to the inlined Redis/Postgres/RabbitMQ.
- **In-chart Deployments** — same pattern as ADR-0020: images in values,
  manifests in `helm/rba/templates`.

Compose stays the no-cluster inner loop and has no scrape stack.

## Decision

1. **Prometheus, Grafana, and kube-state-metrics** ship in the `rba` Helm
   chart (`observability.enabled`, default true). kube-state-metrics is
   **namespace-scoped** (`--namespaces=rba`) so it can chart HPA replica
   counts without cluster-wide RBAC.
2. **PDP exposes `GET /metrics`** (`prometheus-client`). Counters:
   `rba_decisions_total{action,risk_level,fallback}`, histogram
   `rba_risk_score`, plus `http_request_duration_seconds`. This is
   operational scrape, not a `rba-contracts` API.
3. Prometheus scrapes **each PDP pod** via Endpoints `kubernetes_sd` (not the
   ClusterIP). A Service scrape would under-count once the HPA adds replicas.
4. **Grafana Ingress** at `http://localhost:8080/grafana` (Traefik path prefix;
   IdP keeps `/`). Provisioned dashboard **RBA / RBA PDP**. Local credentials
   `admin` / `admin`; anonymous Viewer on.
5. **Load test** is `rba-infra/scripts/load-hpa.sh`: an in-cluster Job of
   `python:3.12-slim` POSTing `/risk/evaluate` with fresh `event_id`s. Hit the
   PDP, not the IdP — the HPA is on the latency-critical service. HPA scale-up
   stabilization is **0s** so a 2–3 minute run can move replicas on a laptop.
6. When the HPA is enabled, the Deployment **omits `spec.replicas`** so Helm
   does not fight the scale subresource.

## Consequences

- Positive: the graded story is runnable (Grafana + HPA replica chart + a
  script that scales 2 → 5). Compose is unchanged.
- Negative: three more pods on the laptop; no Alertmanager; no cAdvisor
  per-pod CPU in Prometheus (HPA CPU still comes from metrics-server;
  kube-state-metrics exposes the HPA's view). Event-lag / worker metrics are
  still later.
- Follow-up: GitOps / CI templates (K8s-3); RabbitMQ / outbox lag; Tilt.
