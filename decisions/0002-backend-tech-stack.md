# ADR-0002: Backend/runtime tech stack

- Status: Accepted
- Date: 2026-08-08

## Context

We need to pick languages/runtimes across the fleet. The decisive constraint is
train/serve parity: the online login path must compute features with the **same
code** used for offline training. That feature code (`rba-features`) is Python. If
the decision service were written in Go/Rust, every feature would have to be
re-implemented in that language — exactly the skew trap the design avoids.

## Decision

- **Backend services: Python 3.12 + FastAPI** (decision-service, admin-api, and the
  async workers). The decision-service imports `rba-features` directly and can load
  the model in-process (a separate sidecar container is optional, for independent
  model deploys).
- **Frontend: React + TypeScript + Vite** (Tailwind + shadcn/ui) for the demo app +
  admin console.
- **Data/messaging: PostgreSQL** (decisions, outbox, profiles history, audit,
  config), **Redis** (online profile cache), **RabbitMQ** first → Kafka optional.
- **ML: LightGBM/XGBoost + scikit-learn + SHAP; MLflow** + object storage (MinIO/S3)
  for the model registry.
- **Platform: Docker, Kubernetes** (k3d/kind local → k3s VM → managed), **Helm** +
  **GitOps (Argo CD/Flux)**, **Tilt/Skaffold** local, **GitHub Actions** CI,
  **Prometheus + Grafana** (+ OpenTelemetry optional).

## Consequences

- Positive: one backend language = far less overhead for a solo dev; parity holds by
  construction; Python is fast enough (I/O-bound path; sub-ms GBDT inference; scale
  via replicas + HPA).
- Negative: Python isn't the raw-throughput leader. Accepted because the login path
  is I/O-bound and horizontally scaled. If a polyglot flourish is wanted, do it on a
  single **off-path** service (e.g. audit) where there's no shared-feature constraint.
- Note: XGBoost/LightGBM cannot use the Apple GPU (CUDA/OpenCL only); irrelevant
  because GBDT is CPU-bound. See ADR-0004 / devlog for the machine decision.
