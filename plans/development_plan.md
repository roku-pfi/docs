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
> **Thesis core (decided — ADR-0015):** **risk-based authentication** —
> explainable, tunable login risk on a real path. That is the research claim.
>
> **Product shell (decided — ADR-0012 / 0014):** a **thesis-scale identity
> platform** (Authentik/Auth0-shaped: users, apps, hosted login, session, MFA,
> admin) that *calls* the PDP. Not a raw risk API and not a carrier-grade IdP.
> Build the shell in stages (ADR-0013); never skip wiring it to RBA (IdP-3).
>
> The architecture will evolve; this plan is structured so that evolution and
> service-splitting are cheap and well-bounded. **This document is the canonical
> roadmap** tools and humans should follow; `status.md` tracks checkbox progress.

---

## 0. TL;DR — what I would actually do

1. **Microservices from the start, deployed on k8s.** Multiple independently
   deployable services, each its own container/pod, orchestrated by Kubernetes, with
   the decision service running **2+ replicas behind an autoscaler**. This is the
   architecture story that earns points.
2. **But protect the login latency budget.** The synchronous **risk** path stays a
   **small number of services** (ideally one, `decision-service`, with model
   inference as an in-pod **sidecar** if we want a separate container). Everything
   *off* that path (audit, profile updates, analytics, training) is its own
   service. Fast login + many services is achieved by **horizontal scaling + async
   off-loading**, not by chopping the PDP into network hops. The thin IdP sits
   *in front* as PEP; it does not absorb the scorer.
3. **Polyrepo — one repo per service** — plus **two library repos** (`rba-features`,
   `rba-contracts`) and one **`rba-infra`** repo (Helm/GitOps). The library repos
   exist specifically to defeat the two biggest polyrepo risks: train/serve feature
   skew and contract drift.
4. **Start the model now.** Begin with the public **Wiefling RBA dataset** (33M
   logins). Its latency/size numbers drive how we size and split services.
5. **Explainability is the thesis.** Transparent, per-signal-explained, tunable
   scoring is the gap vs Okta/Entra/Auth0/Ping. The IdP exists to *deliver* RBA,
   not to replace it (ADR-0015).
6. **Product shell is a thesis-scale IdP (ADR-0012/0014).** Authentik/Auth0-shaped:
   users, apps, hosted login, session, MFA, admin. Do not rewrite the risk core
   into an IAM, and do not implement enterprise protocol suites.
7. **Scope discipline.** Define all service boundaries now; stand up incrementally.
   Cut fancy IAM (federation, SCIM) before cutting k8s/HPA, parity, or leakage.

---

## 1. Guiding principles

- **Scalability & orchestration are deliverables, not afterthoughts.** Stateless
  services, horizontal replicas, HPA, health probes, resource requests/limits,
  rolling deploys. Show the system scale-out under load in Grafana.
- **Explainability is the thesis.** Every decision emits a human-readable reason
  trace with per-signal contributions. The IdP platform exists to deliver that,
  not to hide it (ADR-0015).
- **PDP/PEP split.** The risk system *decides* (`ALLOW / MFA / REAUTH / BLOCK`); the
  **PEP enforces**. Near-term the PEP may be curl, a demo script, or a mock app;
  the **end-product PEP is the thesis-scale IdP platform** (ADR-0012/0014). The PDP stays a vendor-neutral
  REST API either way — never bury identity inside `decision-service`.
- **Additive migration.** Near-term PDP work must remain the October risk backend.
  Prefer new services (`rba-idp`, admin) over reshaping the scorer into an IAM.
- **The risk path is sacred.** Target p95 PDP latency ~100–200 ms. Keep the
  evaluate path to the minimum number of in-pod hops; push everything else async.
  IdP password verify + session are outside that budget but must stay simple.
- **Database-per-service.** Each service owns its data; no shared-table coupling.
  IdP users/sessions live in an IdP DB — not in `rba_decision` / Redis profiles.
- **Contracts are code.** API/event schemas live in `rba-contracts`, versioned,
  with generated clients. No service reaches into another's internals.
- **Train/serve parity via a shared, versioned feature package** (`rba-features`).
- **Config over code.** Rule weights, score→level thresholds, level→action maps are
  versioned config, hot-reloadable, auditable.
- **Time-box ruthlessly.** Define broad, build a complete thin slice first.
  Groups/permissions are a **valid product fit** but **stretch** (after login +
  users + decision browser).

---

## 1.1 Product horizons & migration path (do not skip)

| Horizon | When | What “done” looks like |
|---|---|---|
| **A — PDP core** | done (Phases 1–4 thin slice) | `/risk/evaluate` + async profile/audit. **No demo-kit polish** (ADR-0013). |
| **B — IdP platform** | done (IdP-1…7) | Authentik/Auth0-shaped at thesis scale (ADR-0014): users, apps, hosted login, PDP enforce, session/MFA, admin. |
| **C — Thesis hardening** | K8s-1/2 done | k8s/HPA/observability. K8s-3 (GitOps/CI) **deferred**. Federation/SCIM still out. |
| **D — Product demo** | **now** (ADR-0022) | Client app + scenario control + WebAuthn step-up + country-centroid travel rule. Freeman still decides the score. |

**What near-term work must protect (reuse forever)**

- `rba-features` + parity tests; Freeman serving artifact + `proba_mapping`
- `rba-contracts` `/risk/evaluate` + `rba.decision.made.v1` + policy config
- `decision-service` as **stateless PDP** (Redis read, outbox write)
- Async plane ownership: profile owns Redis writes; audit owns audit store
- `rba-infra` shared data plane

**What not to over-commit before Horizon B**

- Putting passwords, sessions, or user CRUD into `decision-service`
- A polished “final” UI that assumes there will never be an IdP
- OIDC/SAML/SCIM, org directories, or enterprise permission engines (ADR-0014)
- Treating Horizon A’s demo PEP as the product architecture

**Migration in one line:** the PDP is the brain; the IdP platform is the Auth0-like
face that *calls* the brain — same risk contracts, new product shell.

---

## 2. Target architecture

### 2.1 System topology (Kubernetes)

```mermaid
flowchart TB
  CLIENT[Demo client app] -->|redirect / login| IDP

  subgraph K8S["Kubernetes cluster"]
    ING[Ingress / API Gateway]
    ING --> IDP[rba-idp<br/>thin IdP = PEP<br/>users · session · MFA challenge]
    ING --> FE[frontend<br/>IdP pages + admin console]
    ING --> ADMIN[admin-api<br/>users · policy · decisions<br/>groups/permissions stretch]
    IDP -->|POST /risk/evaluate| DEC

    subgraph POD["decision-service Pod (2+ replicas, HPA)"]
      DEC["decision-service (PDP)<br/>feature assembly + rules + policy + explain<br/>STATELESS, latency-critical"]
      SIDE["model-inference sidecar<br/>(same pod, localhost/UDS)"]
      DEC <-->|no network hop| SIDE
    end

    DEC -->|read-only, O(1)| REDIS[(Redis<br/>online profiles)]
    DEC -->|decision + outbox<br/>one txn| PGDEC[(Postgres<br/>decision-service)]
    IDP --> PGIDP[(Postgres<br/>idp users/sessions)]

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

Horizon A demos may call `DEC` directly (curl / script). Horizon B always goes
`CLIENT → IDP → DEC`.

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
| `decision-service` (PDP) | Validate request, assemble features (via `rba-features`), run rules, combine scores, apply policy, produce explanation, write decision+outbox | decision DB (Postgres) | **HPA, 2+ replicas** | The hot **risk** path. Kept as one service on purpose. **No identity store here.** |
| `model-inference` (sidecar) | Load model from registry, `predict_proba` | model artifact (read-only) | Scales with its pod | Separate container for independent model deploys **without** a network hop. |
| `rba-idp` (PEP, Horizon B) | Login UI/API, local user verify, call PDP, enforce MFA/block, issue session | IdP DB (users, sessions, MFA state) | 1–2 replicas | Product shell. Owns demo-client login; never inlines Freeman. |
| `admin-api` | Users (proxy/CRUD toward IdP data or shared admin read models), policies, thresholds, model activation, decision browser; **groups/permissions stretch** | config DB (+ reads from audit/IdP as needed) | 1–2 replicas | Control plane vs data plane; not on PDP latency path. |

> **Lean vs fine-grained call:** I recommend keeping feature-assembly, rules, and
> policy **in-process** inside `decision-service`, and only breaking out **model
> inference as a sidecar**. That is the "mix" that maximizes login speed while still
> giving you multiple containers on the request path. If you later insist on a
> standalone `feature-service` or `policy-service`, do it as a **sidecar** too (same
> pod) so latency stays flat — never as a cross-node call.
>
> **IdP vs PDP:** password verify and session cookies may add wall-clock time to the
> *user* login, but they must not be merged into the PDP process. The evaluate call
> stays a tight RPC.

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
| `rba-idp` + `frontend` | Horizon B: login / MFA / session UX; admin console (users, decisions, policy; groups/permissions stretch). Horizon A may skip UI. |
| Observability | Prometheus, Grafana, structured JSON logs (Loki optional) |

### 3.4 Splitting sequence (define all now, stand up incrementally)

1. `decision-service` (+ inline model first, sidecar later) → `event-publisher` +
   `profile-service` + `audit-service` (Phases 3–4 — **done / in progress**).
2. Swap in-process bus → **RabbitMQ**, add `analytics-service`, `dataset-builder`,
   `ml-training` Job.
3. Split out `model-inference` as a sidecar; add **`rba-idp`** + `admin-api` +
   `frontend` (Horizon B thin IdP).
4. Stretch: groups/permissions in admin/IdP; (optional) `alerting-service`, Kafka,
   service mesh.

---

## 4. Repository strategy (polyrepo)

One repo per running service, **plus** shared library and infra repos. The library
repos are non-negotiable — they're what keep polyrepo from causing skew.

### 4.1 Repo catalog

```text
# Running services (one repo each → one image → one Deployment/pod)
rba-decision-service
rba-model-inference
rba-idp                     # thin IdP (Horizon B) — PEP; local users/sessions/MFA
rba-admin-api               # policy + decision browser + user admin; groups stretch
rba-event-publisher
rba-profile-service
rba-audit-service
rba-analytics-service
rba-alerting-service        # optional
rba-dataset-builder
rba-ml-training
rba-frontend                # IdP pages + admin console (may start colocated with idp)

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
3. **Deterministic rules for hard overrides** (ADR-0022): country-centroid
   **impossible travel** may **escalate** the policy action (default ALLOW →
   `REQUIRE_MFA`, not BLOCK). VPN/hosting ASNs skip the physics check and emit
   `vpn_or_hosting` instead. Rules do not replace Freeman’s `risk_score`; they
   can raise the action when physics (or a hard policy) says the score is not
   enough. GPS / city GeoIP / Freeman-categorical travel stay out.

Model outputs a probability; **policy engine → action** (score→level and
level→action per app-sensitivity are versioned config). The demo app never
calls Freeman — the IdP asks the PDP (ADR-0022).

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

**Not the same as Demo-2.** The live demo uses a **scenario control** (next-login
context + a seeded home profile). The Phase 6 generator remains optional later.

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

Model + contracts first; then the request path; then async services; then k8s /
observability; **thin IdP + admin as the Horizon B product shell**; **Horizon D
product demo** (ADR-0022) before more GitOps; report last.
See §1.1 and ADR-0012 for migration rules.

### Phase 0 — Foundations & repos (≈1–1.5 weeks)
- Bootstrap `rba-infra` (local cluster via k3d/kind, Tilt/Skaffold, CI template,
  Prometheus/Grafana). Create `rba-contracts` + `rba-features` skeletons.
- Stand up `rba-decision-service` repo with a stub `/risk/evaluate`.
- *(Compose Redis/Postgres/RabbitMQ + k3d/Helm local stack landed, ADR-0020;
  Tilt / GitOps / CI templates still open.)*

### Phase 1 — Data & model feasibility (≈2–3 weeks) ← **done**
- Wiefling subset + EDA. Implement `rba-features` + offline replay driver.
- Baselines: Freeman, LogReg, RF, GBM. Run the leakage comparison.
- Chronological split; record RBA metrics **and inference latency + model size**
  (these size the sidecar and the pods).

### Phase 2 — Freeze contracts (≈1 week) ← **done**
- Lock feature schema, model interface, `/risk/evaluate` contract, event contract in
  `rba-contracts`. Define score→level / level→action config format.

### Phase 3 — Request path (≈3–4 weeks) ← **done** (local k8s deferred)
- `rba-decision-service`: validation, Redis profile read, features, rules, model
  (inline first), policy, explanation, decision+outbox write. Parity tests green.
  Fallback behavior on model/service failure. Containerize; run on local cluster.

### Phase 4 — Async services (≈2–3 weeks) ← **thin slice done**
- `rba-event-publisher` (outbox drainer) → RabbitMQ. `rba-profile-service`,
  `rba-audit-service` consumers. Idempotency via `event_id`. Each its own pod.

### Phase 5 — Observability & scenario/load testing (≈2 weeks)
- Prometheus metrics per stage; Grafana dashboards (decision mix, score dist,
  p95/p99 latency, event lag). Scenario + **load tests that trigger the HPA** — this
  is your scalability demo.

### Phase 6 — ML lifecycle + generator (≈2 weeks)
- `rba-dataset-builder` + `rba-ml-training` Job/CronJob + MLflow registry. Split
  `rba-model-inference` out as a **sidecar**. Build the synthetic generator.

### Phase 7 — IdP platform + admin + k8s hardening ← **Horizon B** (ADR-0012/0013/0014)

One stage at a time. Do not start stage *n+1* until *n* is checked in `status.md`.
Product shape: thesis-scale Authentik/Auth0, not a protocol IdP.

| Stage | What ships | Explicitly not yet |
|---|---|---|
| **IdP-1** Contracts | `rba-contracts` 0.2.0: login / MFA / session / public user. PDP API unchanged. | No `rba-idp` repo |
| **IdP-2** Identity store | New `rba-idp` + `rba_idp` DB: users, password verify, **seeded application** (registered client). | No PDP call, no session, no UI, no OIDC |
| **IdP-3** PDP enforce | IdP calls `/risk/evaluate`; maps action → login outcome; returns reasons. | No cookie, no OTP |
| **IdP-4** Session + mock MFA | Token/cookie on `ALLOW`; challenge + mock OTP for MFA/REAUTH; `BLOCK` rejected. | No HTML UI |
| **IdP-5** Hosted login | Login page on the IdP (apps send users here). | No admin console |
| **IdP-6** Admin console | Users, applications, decisions, policy. | No groups |
| **IdP-7** Stretch | Groups / app-scoped permissions. | Federation / SCIM / full OIDC still out |

k8s hardening (HPA, probes, secrets, GitOps) stays in this phase but is **not**
gated on IdP-7. **K8s-1 and K8s-2 are done.** K8s-3 (GitOps/CI/Tilt) is
**deferred** until Horizon D exists (ADR-0022).

### Horizon D — Product demo (ADR-0022) ← **current**

One stage at a time. Canonical checkboxes: `status.md`. Do not skip Demo-1.

| Stage | What ships | Explicitly not |
|---|---|---|
| **Demo-1** Signals + travel rule | Country on the login path; `impossible_travel` in `rba-features` (country centroids); PDP escalates ALLOW → MFA; VPN/hosting skip. Parity green. | GPS, city GeoIP, Freeman travel categorical |
| **Demo-2** Scenarios + seed | Seeded usual profile; next-login context picker (home / new country / teleport / VPN). | Phase 6 generator, extra worker |
| **Demo-3** Client app | Banking UI on `rba-idp` `/app` (colocated). After `AUTHENTICATED`, a normal app home — **no** score/reasons (ADR-0023). Optional forum app (existing looser policy). | New repo/service, OIDC |
| **Demo-4** Real step-up | WebAuthn passkey for `REQUIRE_MFA` (opaque copy, ADR-0023). Mock OTP for tests. No re-score after MFA. | TOTP/SMS/push unless WebAuthn is blocked |

Walkthrough: home ALLOW → app (opaque); novel country → generic MFA
(Freeman+policy); teleport → generic MFA (rule); VPN → generic MFA as untrusted
network; **admin Decisions** shows why.

### Phase 8 — Report & defense buffer (ongoing)
- Feed the thesis continuously: EDA plots, metric tables, architecture + k8s
  diagrams, leakage discussion, scale-out screenshots, threat model, **product
  demo walkthrough** (Horizon D).

> If time gets tight, cut in this order: service mesh → Kafka (RabbitMQ is enough) →
> alerting-service → synthetic generator → K8s-3 GitOps/CI → TOTP/extra factors →
> polished admin chrome. **Do not** cut: k8s + HPA (already shipped),
> `rba-features` parity, leakage experiment, PDP `/risk/evaluate`, the IdP→PDP
> split, or Horizon D Demo-1…3 (a scored login into an app with visible reasons).
> Groups/permissions already shipped (IdP-7).

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
| Scope creep on a solo deadline | Explicit cut order; Horizon D stays thin (ADR-0022); no extra factor products |
| Near-term demo locks wrong shape | Additive migration: IdP wraps PDP; no identity in decision-service (ADR-0012) |

---

## 12. What I'd do *this week*

Horizon D, starting at **Demo-2** (ADR-0022) — seed + scenario picker so the
walkthrough is not “every user looks new”:

1. Seed `demo@example.com` with a usual home profile (AR / residential ASN).
2. Demo-only control to pick the next login context (home / new country /
   impossible travel / VPN). Presenter-only, not customer chrome (ADR-0023).
3. Only then Demo-3 (banking `/app`) and Demo-4 (WebAuthn).

---

### References
- Wiefling, Jørgensen, Thunem, Lo Iacono — *Pump Up Password Security! …*, ACM TOPS
  (2022). Dataset: Zenodo 6782156 / Kaggle `dasgroup/rba-dataset` /
  `github.com/das-group/rba-dataset`.
- Freeman et al. — *Who Are You? A Statistical Approach to Measuring User
  Authenticity*, NDSS (2016). (Explainable likelihood-ratio RBA model.)
- *A hybrid ML and explainable AI framework for optimizing RBA*, PLOS One (2025) —
  GBM/XGBoost + SHAP on the same dataset (related work for the explainability angle).
