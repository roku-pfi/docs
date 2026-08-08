# Adaptive / Risk-Based Authentication — Development Plan (my proposal)

> My opinionated plan for building the system. Distinct from the exploratory
> discussion in `material_sources/rba_architecture_conversation.md`.
>
> **Target shape (decided):** a **polyrepo** set of **containerized microservices**
> deployed as **pods under Kubernetes**, where the **login decision path is kept lean
> and horizontally scalable** so authentication stays fast. Scalability and
> orchestration are treated as first-class, graded deliverables — a design that only
> runs as a single un-scalable monolith is explicitly a non-goal.
>
> The architecture will evolve; this plan is structured so that evolution and
> service-splitting are cheap and well-bounded.

---

## 0. TL;DR — what I would actually do

1. **Microservices from the start, deployed on k8s.** Multiple independently
   deployable services, each its own container/pod, orchestrated by Kubernetes, with
   the decision service running **2+ replicas behind an autoscaler**. This is the
   architecture story that earns points.
2. **But protect the login latency budget.** The synchronous decision path stays a
   **small number of services** (ideally one, `decision-service`, with model
   inference as an in-pod **sidecar** if we want a separate container). Everything
   *off* the request path (audit, profile updates, analytics, training) is its own
   service. Fast login + many services is achieved by **horizontal scaling + async
   off-loading**, not by chopping the request path into network hops.
3. **Polyrepo — one repo per service** — plus **two library repos** (`rba-features`,
   `rba-contracts`) and one **`rba-infra`** repo (Helm/GitOps). The library repos
   exist specifically to defeat the two biggest polyrepo risks: train/serve feature
   skew and contract drift.
4. **Start the model now.** Begin with the public **Wiefling RBA dataset** (33M
   logins). Its latency/size numbers drive how we size and split services.
5. **Explainability is the product.** Transparent, per-signal-explained, tunable
   scoring is your market gap (Okta/Entra/Auth0/Ping hide theirs). Build it in from
   line one.
6. **Scope discipline for a solo December deadline.** Define all service boundaries
   now, but *stand up* services incrementally; demo frontend is last.

---

## 1. Guiding principles

- **Scalability & orchestration are deliverables, not afterthoughts.** Stateless
  services, horizontal replicas, HPA, health probes, resource requests/limits,
  rolling deploys. Show the system scale-out under load in Grafana.
- **Explainability is the product.** Every decision emits a human-readable reason
  trace with per-signal contributions.
- **PDP/PEP split.** The risk system *decides* (`ALLOW / MFA / REAUTH / BLOCK`); the
  caller *enforces*. Stays vendor-neutral, callable over REST.
- **The login path is sacred.** Target p95 internal latency ~100–200 ms. Keep the
  request path to the minimum number of in-pod hops; push everything else async.
- **Database-per-service.** Each service owns its data; no shared-table coupling.
  (Another architecture-grading win, and it makes the pods truly independent.)
- **Contracts are code.** API/event schemas live in `rba-contracts`, versioned,
  with generated clients. No service reaches into another's internals.
- **Train/serve parity via a shared, versioned feature package** (`rba-features`).
- **Config over code.** Rule weights, score→level thresholds, level→action maps are
  versioned config, hot-reloadable, auditable.
- **Time-box ruthlessly.** Define broad, build a complete thin slice first.

---

## 2. Target architecture

### 2.1 System topology (Kubernetes)

```mermaid
flowchart TB
  CLIENT[Demo app / API tests / IdP plugin<br/>= PEP] -->|HTTPS| ING[Ingress / API Gateway<br/>Traefik or Kong]

  subgraph K8S["Kubernetes cluster"]
    ING --> FE[frontend<br/>demo app + admin console]
    ING --> ADMIN[admin-api<br/>policies, thresholds, model activation]
    ING -->|POST /risk/evaluate| DEC

    subgraph POD["decision-service Pod (2+ replicas, HPA)"]
      DEC["decision-service (PDP)<br/>feature assembly + rules + policy + explain<br/>STATELESS, latency-critical"]
      SIDE["model-inference sidecar<br/>(same pod, localhost/UDS)"]
      DEC <-->|no network hop| SIDE
    end

    DEC -->|read-only, O(1)| REDIS[(Redis<br/>online profiles)]
    DEC -->|decision + outbox<br/>one txn| PGDEC[(Postgres<br/>decision-service)]

    PUB[event-publisher<br/>drains outbox] --> BUS[(Event bus<br/>RabbitMQ → Kafka)]
    PGDEC --> PUB

    BUS --> PROF[profile-service<br/>consumer, owns profiles]
    BUS --> AUD[audit-service<br/>owns audit store]
    BUS --> ANA[analytics-service<br/>metrics/aggregations]
    BUS --> ALR[alerting-service optional]

    PROF --> REDIS
    PROF --> PGPROF[(Postgres<br/>profile history)]
    AUD --> PGAUD[(Postgres / object store<br/>audit)]

    subgraph ML["ML plane (Jobs/CronJobs, not on request path)"]
      TRAIN[ml-training Job] --> REG[(Model registry<br/>MLflow + object store)]
      DSB[dataset-builder] --> TRAIN
    end
    BUS --> DSB
    REG -.image/artifact pulled at startup.-> SIDE

    OBS[Observability<br/>Prometheus + Grafana + logs]
  end
```

### 2.2 How we stay fast *and* microservice-y (the core trade-off you raised)

You want maximum login performance **and** a real multi-service, orchestrated
architecture. Those aren't in conflict if we use the right tools for "many
services" instead of splitting the hot path:

- **Horizontal scale, not path-splitting.** `decision-service` is stateless →
  run N replicas behind an HPA. That is the scalability the grader wants, with **zero
  added per-request latency**. Splitting features/scoring/policy into separate
  network services would add hops to the one path that must stay fast — so we don't.
- **Sidecar for model inference.** If we want model serving to be its *own container*
  (nice for architecture points and independent model updates), run it as a
  **sidecar in the same pod**. Communication is over `localhost`/Unix socket — no
  cross-node network latency, but it's still a separate, independently-versioned
  container.
- **Async off-loading.** Audit, profile updates, analytics, alerting, dataset
  building all happen **after** the response, via the event bus. They can be as
  fine-grained as we like without touching login latency.
- **Redis for the read path.** Profiles are materialized ahead of time; the decision
  path does an O(1) key lookup, never a historical scan.
- **Cached enrichment only.** Any IP-reputation / geo / ASN data is served from a
  local cache refreshed asynchronously — never a blocking external call at login.

Net: the request path is **1 pod (with a sidecar)**; the rest of the system is a
proper fleet of services and jobs.

---

## 3. Service decomposition (defined now, refine later)

Classification by whether a service sits **on the synchronous login path** (must be
fast, minimal count) or **off it** (free to be fine-grained).

### 3.1 Request-path services (keep minimal — this is where latency lives)

| Service | Responsibility | Owns (data) | Scaling | Split rationale |
|---|---|---|---|---|
| `decision-service` (PDP) | Validate request, assemble features (via `rba-features`), run rules, combine scores, apply policy, produce explanation, write decision+outbox | decision DB (Postgres) | **HPA, 2+ replicas** | The hot path. Kept as one service on purpose. |
| `model-inference` (sidecar) | Load model from registry, `predict_proba` | model artifact (read-only) | Scales with its pod | Separate container for independent model deploys **without** a network hop. |
| `admin-api` | CRUD policies, thresholds, rule weights; activate model versions; read decisions | config DB | 1–2 replicas | Not latency-critical, but must be separate from the PDP (control plane vs data plane). |

> **Lean vs fine-grained call:** I recommend keeping feature-assembly, rules, and
> policy **in-process** inside `decision-service`, and only breaking out **model
> inference as a sidecar**. That is the "mix" that maximizes login speed while still
> giving you multiple containers on the request path. If you later insist on a
> standalone `feature-service` or `policy-service`, do it as a **sidecar** too (same
> pod) so latency stays flat — never as a cross-node call.

### 3.2 Off-path services (free to be as granular as you want)

| Service | Sync/Async | Responsibility | Owns (data) |
|---|---|---|---|
| `event-publisher` | async | Drain the `outbox` table → publish to the bus (transactional-outbox bridge; keeps B-path independent of broker uptime) | — |
| `profile-service` | async consumer | Consume decision events, update materialized online profiles + history | Redis (profiles) + Postgres (history) |
| `audit-service` | async consumer | Persist full decision/audit records (signals, features, rules fired, score, action, versions) | Postgres / object store |
| `analytics-service` | async consumer | Aggregations, score distributions, decision-mix metrics | analytics DB / Prometheus |
| `alerting-service` (optional) | async consumer | Fire alerts on suspicious patterns | — |
| `dataset-builder` | batch/consumer | Turn stored events into labeled training rows (uses `rba-features`) | dataset store |
| `ml-training` | k8s Job/CronJob | Train + evaluate baselines and GBM; push artifacts | — |
| `model-registry` | platform | Version + store model artifacts + metadata (use **MLflow** + object storage; don't build your own) | object store |

### 3.3 Edge / platform / UI

| Service | Responsibility |
|---|---|
| `ingress` / API gateway | TLS termination, routing, rate limiting, auth of callers (Traefik or Kong; config lives in `rba-infra`) |
| `frontend` | Demo app (login → shows ALLOW/MFA/BLOCK, mock OTP) + admin console UI |
| Observability | Prometheus, Grafana, structured JSON logs (Loki optional) |

### 3.4 Splitting sequence (define all now, stand up incrementally)

1. `decision-service` (+ inline model first, sidecar later) → `event-publisher` +
   `profile-service` + `audit-service` (in-process bus initially).
2. Swap in-process bus → **RabbitMQ**, add `analytics-service`, `dataset-builder`,
   `ml-training` Job.
3. Split out `model-inference` as a sidecar; add `admin-api`; add `frontend`.
4. (Optional) `alerting-service`, migrate bus → **Kafka**, add a service mesh
   (Linkerd/Istio) for mTLS + traffic metrics if you want extra architecture points.

---

## 4. Repository strategy (polyrepo)

One repo per running service, **plus** shared library and infra repos. The library
repos are non-negotiable — they're what keep polyrepo from causing skew.

### 4.1 Repo catalog

```text
# Running services (one repo each → one image → one Deployment/pod)
rba-decision-service
rba-model-inference
rba-admin-api
rba-event-publisher
rba-profile-service
rba-audit-service
rba-analytics-service
rba-alerting-service        # optional
rba-dataset-builder
rba-ml-training
rba-frontend

# Shared libraries (published, versioned packages — NOT deployed)
rba-features                # the feature library; imported by decision-service,
                            # dataset-builder, ml-training. Prevents train/serve skew.
rba-contracts               # OpenAPI + AsyncAPI/proto schemas + generated clients.
                            # Prevents contract drift between services.

# Platform
rba-infra                   # Helm charts, k8s manifests, GitOps (ArgoCD/Flux),
                            # local dev (Tilt/Skaffold or compose), gateway config,
                            # Prometheus/Grafana provisioning, CI templates.
```

### 4.2 Standard per-service repo layout

```text
rba-<service>/
├── src/                    # service code (hexagonal: api / core / ports / adapters)
├── tests/                  # unit + contract tests
├── Dockerfile
├── helm/ or k8s/           # or reference charts in rba-infra
├── openapi.yaml            # this service's contract (source of truth pushed to rba-contracts)
├── .github/workflows/      # CI: test → build image → push to registry → bump GitOps
└── README.md
```

### 4.3 The polyrepo tax, and how we pay it deliberately

Polyrepo's real costs for a solo dev — and the concrete mitigation each:

- **Train/serve feature skew** → `rba-features` published with semver; both
  `decision-service` and `ml-training`/`dataset-builder` pin a version. A **parity
  test suite** (same inputs → identical vectors offline vs online) runs in CI on
  every `rba-features` release. *This is the single most important safeguard in the
  whole project.*
- **Contract drift** → `rba-contracts` holds the API/event schemas; services import
  generated clients. **Consumer-driven contract tests** (e.g., Pact) fail CI when a
  producer breaks a consumer.
- **N× CI/CD** → one reusable CI template in `rba-infra`; every repo builds an image
  to one container registry, then updates a GitOps manifest → ArgoCD/Flux syncs the
  cluster. This *is* the deployment story that scores architecture points.
- **Painful local dev across many repos** → **Tilt or Skaffold** in `rba-infra`
  builds/watches all services into a local cluster (k3d/kind); fall back to a
  `docker-compose` for the subset you're actively hacking on.
- **Version coordination** → keep contracts backward-compatible; roll producers
  before consumers; use the event bus to decouple.

---

## 5. Train/serve parity via `rba-features` (the decision that matters most)

A feature (`country_seen_before`, `impossible_travel`, …) must be computed
identically offline and online, or the model silently rots. Recipe:

- Each feature is a pure function `f(current_event, profile_state) -> value` in
  `rba-features`.
- **Offline** (`dataset-builder`/`ml-training`): replay events per user in timestamp
  order, maintaining `profile_state`, calling `f`. This mechanically enforces
  "past-only" information → no leakage.
- **Online** (`decision-service`): load materialized `profile_state` from Redis, call
  the same `f`.
- **Parity CI:** N events through both paths → assert equal vectors.

Initial features (all from the Wiefling dataset): `device_type/os/browser_seen_before`,
`ip/asn/country/region/city_seen_before`, `unusual_login_hour`,
`seconds_since_last_login`, `distance_from_last_login_km`, `impossible_travel`,
`rtt_deviation_from_user_mean`, `failed_logins_last_hour`, `user_login_count`
(history maturity), and the leakage-sensitive `is_attack_ip` (see §7).

---

## 6. Scoring strategy (where your differentiator lives)

Three layers, combined by the policy engine inside `decision-service`:

1. **Freeman et al. (2016) likelihood-ratio scorer — the explainable baseline.**
   Risk = product of per-feature `P(value|attacker)/P(value|legit)`. Every feature
   contributes an inspectable multiplier → native per-signal explanations, fully
   tunable. This is exactly the transparency gap you identified vs. the vendors, and
   it can power a credible MVP with no ML at all.
2. **ML enhancement — Gradient Boosting (LightGBM/XGBoost) as primary**, with
   **Logistic Regression + Random Forest** as reference baselines. GBM + **SHAP**
   gives per-decision explanations (a recent PLOS One RBA paper uses GBM/XGBoost +
   SHAP on this same dataset — good related work). Keep RF since your proposal names
   it; report both.
3. **Deterministic rules for hard overrides** (disabled user → BLOCK; critical app +
   unknown device → force MFA). Rules produce evidence/adjustments, not the final
   action.

Model outputs a probability; **policy engine → action** (score→level and
level→action per app-sensitivity are versioned config).

**Mandatory leakage experiment:** train Variant A (with `is_attack_ip`) and Variant B
(behavioral/context only). If A ≫ B, the model is memorizing bad IPs, not learning
behavior. B is the honest number for the thesis.

---

## 7. Data strategy

### 7.1 Backbone — public Wiefling RBA dataset (start here, now)

- **What:** *Login Data Set for Risk-Based Authentication* — synthesized from 3.3M
  users / ~33M logins at a Norwegian SSO (Feb 2020–Feb 2021). Purpose-built for RBA;
  safe to publish results on.
- **Where:** Zenodo `zenodo.org/records/6782156` · Kaggle `dasgroup/rba-dataset` ·
  GitHub `das-group/rba-dataset`. ~1.1 GB zipped / ~9 GB raw → start with a per-user
  stratified subset.
- **Fields:** IP, Country/Region/City/ASN, User Agent, OS + Browser (name+version),
  Device Type, User ID, Login Timestamp, RTT(ms), Login Successful, **Is Attack IP**,
  **Is Account Takeover** (label).
- **Design-around gotchas:** extreme class imbalance (use PR-AUC, recall@low-FPR),
  sparse history (mean 3.8 / median 2 logins per user → report metrics by history
  depth), `Is Attack IP` leakage (§6), and temporal integrity (**chronological
  split** 70/15/15, never random).

### 7.2 Secondary datasets

The Wiefling set is the only public one that natively matches your signal space.
Network-intrusion sets (CIC-IDS/UNSW-NB15) or behavioral-biometric sets (HuMIdb) are
inspiration only — don't burn time hunting for a second real login dataset.

### 7.3 Synthetic generator — after EDA, to *complement* not replace

Build it once EDA shows what's underrepresented, to inject attacks the public set
lacks and the `application_sensitivity` dimension it doesn't have:

- Impossible travel, credential-stuffing bursts, MFA-fatigue/repeated-failure chains,
  new-device-familiar-location vs. known-device-new-country, per-app-sensitivity mixes.

Design it as a **profile-driven simulator**: define user archetypes (usual
devices/countries/hours), sample "normal" logins, splice in parameterized attack
episodes with ground-truth labels, emit rows in the **same schema** as the real set
so both flow through `rba-features`. Always tag `data_source ∈ {real, synthetic}` and
report metrics split by source. Lives in `rba-ml-training` (or its own repo if it
grows).

---

## 8. Development roadmap (phased, time-boxed to December)

Model + contracts first; then the request path; then async services; then k8s
hardening; frontend last.

### Phase 0 — Foundations & repos (≈1–1.5 weeks)
- Bootstrap `rba-infra` (local cluster via k3d/kind, Tilt/Skaffold, CI template,
  Prometheus/Grafana). Create `rba-contracts` + `rba-features` skeletons.
- Stand up `rba-decision-service` repo with a stub `/risk/evaluate`.

### Phase 1 — Data & model feasibility (≈2–3 weeks) ← **start now**
- Wiefling subset + EDA. Implement `rba-features` + offline replay driver.
- Baselines: Freeman, LogReg, RF, GBM. Run the leakage comparison.
- Chronological split; record RBA metrics **and inference latency + model size**
  (these size the sidecar and the pods).

### Phase 2 — Freeze contracts (≈1 week)
- Lock feature schema, model interface, `/risk/evaluate` contract, event contract in
  `rba-contracts`. Define score→level / level→action config format.

### Phase 3 — Request path (≈3–4 weeks)
- `rba-decision-service`: validation, Redis profile read, features, rules, model
  (inline first), policy, explanation, decision+outbox write. Parity tests green.
  Fallback behavior on model/service failure. Containerize; run on local cluster.

### Phase 4 — Async services (≈2–3 weeks)
- `rba-event-publisher` (outbox drainer) → RabbitMQ. `rba-profile-service`,
  `rba-audit-service` consumers. Idempotency via `event_id`. Each its own pod.

### Phase 5 — Observability & scenario/load testing (≈2 weeks)
- Prometheus metrics per stage; Grafana dashboards (decision mix, score dist,
  p95/p99 latency, event lag). Scenario + **load tests that trigger the HPA** — this
  is your scalability demo.

### Phase 6 — ML lifecycle + generator (≈2 weeks)
- `rba-dataset-builder` + `rba-ml-training` Job/CronJob + MLflow registry. Split
  `rba-model-inference` out as a **sidecar**. Build the synthetic generator.

### Phase 7 — k8s hardening + frontend + admin (≈2 weeks)
- HPA, liveness/readiness probes, resource requests/limits, rolling deploys,
  secrets, network policies, GitOps sync. `rba-admin-api` + `rba-frontend` (minimal
  PEP with mock OTP).

### Phase 8 — Report & defense buffer (ongoing)
- Feed the thesis continuously: EDA plots, metric tables, architecture + k8s
  diagrams, leakage discussion, scale-out screenshots, threat model.

> If time gets tight, cut in this order: service mesh → Kafka (RabbitMQ is enough) →
> alerting-service → synthetic generator → fancy frontend. **Do not** cut: k8s +
> HPA (that's the graded part), `rba-features` parity, the leakage experiment.

---

## 9. Infrastructure & Kubernetes specifics

Staged so you always have something running:

- **Local (now):** `k3d`/`kind` single-node cluster driven by **Tilt/Skaffold** from
  `rba-infra`; Postgres/Redis/RabbitMQ via Helm; Prometheus + Grafana. Train in a
  local venv/Job.
- **Integrated (mid):** a single Linux VM running **k3s** (lightweight k8s) — real
  orchestration, cheap. Ingress public; datastores/brokers/Grafana private. This
  already satisfies "pods managed by an orchestrator."
- **Showcase (end, optional):** a managed cluster (GKE/EKS/DO) or multi-node k3s for
  the defense; keep stateful services managed/external.

k8s features to implement (these *are* the grade): **Deployments + HPA** on
`decision-service`, **liveness/readiness/startup probes**, **resource
requests/limits**, **ConfigMaps/Secrets**, **rolling updates**, **NetworkPolicies**
(only ingress + gateway public), **Jobs/CronJobs** for training, **GitOps**
(ArgoCD/Flux) from `rba-infra`, optional **service mesh** (Linkerd) for mTLS +
traffic metrics.

**Train vs. infer live apart:** inference = small, low-latency, always-on
(Deployment + HPA); training = bursty, heavy, offline (Job/CronJob, separate
resource quota, ideally its own node pool). They must never compete for the request
path's resources.

---

## 10. Evaluation metrics

**Model (report these, not accuracy):** account-takeover **recall @ fixed low FPR**
(e.g. ≤1%), **PR-AUC** + ROC-AUC, **challenge rate** (% legit logins forced into
MFA), **logins-to-protection** (history needed before RBA helps — the Wiefling
paper's key insight), per-history-depth breakdown, inference latency + model size,
and always **Variant A vs B**.

**Architecture/scalability (the graded story):** p95/p99 login latency under load,
sustained throughput (req/s), **autoscaling behavior** (replica count vs. load),
event-processing lag, error/timeout rates, graceful-degradation behavior when a
dependency is down.

---

## 11. Top risks & defenses

| Risk | Defense baked into the plan |
|---|---|
| Over-splitting adds login latency | Request path = 1 service + sidecar; fine-graining only off-path (§2.2, §3.1) |
| Train/serve skew (worsened by polyrepo) | `rba-features` versioned package + parity CI (§4.3, §5) |
| Contract drift across repos | `rba-contracts` + consumer-driven contract tests (§4.3) |
| Polyrepo CI/infra overhead sinks the timeline | Reusable CI template + GitOps in `rba-infra`; incremental service standup (§4.3, §3.4) |
| Label leakage via attack-IP | Mandatory A/B model comparison (§6, §7.1) |
| "Black box" scoring (kills your USP) | Freeman explainable core + SHAP + reason traces (§6) |
| Distributed debugging pain | Structured logs + correlation `event_id` + (optional) mesh tracing |
| Scope creep on a solo deadline | Explicit cut order; frontend last (§8) |

---

## 12. What I'd do *this week*

1. Bootstrap `rba-infra` with a local k3d/kind cluster + Tilt, and create the
   `rba-decision-service`, `rba-features`, `rba-contracts` repos (empty but wired to
   CI).
2. Pull a stratified Wiefling subset; run EDA (imbalance, logins/user, missingness,
   time span).
3. Implement 3–4 features in `rba-features` + the offline replay driver; train a
   **Freeman baseline** and a **GBM** with a chronological split.
4. Run the with/without-`is_attack_ip` comparison; record recall@1%FPR, PR-AUC,
   challenge rate, and **inference latency** (this sizes the sidecar/pod).

This answers "should I train now?" (yes) and produces the numbers that let us lock
service boundaries and pod sizing with evidence.

---

### References
- Wiefling, Jørgensen, Thunem, Lo Iacono — *Pump Up Password Security! …*, ACM TOPS
  (2022). Dataset: Zenodo 6782156 / Kaggle `dasgroup/rba-dataset` /
  `github.com/das-group/rba-dataset`.
- Freeman et al. — *Who Are You? A Statistical Approach to Measuring User
  Authenticity*, NDSS (2016). (Explainable likelihood-ratio RBA model.)
- *A hybrid ML and explainable AI framework for optimizing RBA*, PLOS One (2025) —
  GBM/XGBoost + SHAP on the same dataset (related work for the explainability angle).
