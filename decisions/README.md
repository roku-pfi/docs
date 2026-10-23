# Architecture Decision Records (ADRs)

Each ADR captures one decision: the context that forced it, what we decided, and
the consequences. ADRs are immutable once **Accepted** — if we change our mind, we
add a new ADR that **supersedes** the old one (and mark the old one accordingly),
rather than editing history. This preserves the reasoning trail for the thesis.

Read [`status.md`](../plans/status.md) for *where we are*; this folder is *why
we chose this shape*. Numbering is chronological. Next unused number: **0038**.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-repo-and-deployment-strategy.md) | Repository & deployment strategy (polyrepo + microservices + Kubernetes) | Accepted |
| [0002](0002-backend-tech-stack.md) | Backend/runtime tech stack | Accepted |
| [0003](0003-dataset-selection-and-synthetic-data.md) | Dataset selection & use of synthesized data | Accepted |
| [0004](0004-modelling-and-label-strategy.md) | Modelling approach & label strategy | Accepted |
| [0005](0005-sentinel-user-exclusion.md) | Data cleaning: excluding non-human sentinel accounts | Accepted |
| [0006](0006-ai-tooling-rules-skill-mcp.md) | AI-assisted development tooling (rules, skill, MCP) | Accepted |
| [0007](0007-evaluation-protocol.md) | Evaluation protocol (label-covered window + chronological split) | Accepted |
| [0008](0008-contracts-freeze.md) | Freeze feature / model / PDP / event / policy contracts | Accepted |
| [0009](0009-online-profile-freeman-serving.md) | Online ProfileState counts + Freeman JSON serving | Accepted |
| [0010](0010-shared-local-data-plane.md) | Shared local data plane lives in `rba-infra` | Accepted |
| [0011](0011-async-outbox-rabbitmq.md) | Phase 4 async plane — outbox → RabbitMQ → profile/audit | Accepted |
| [0012](0012-thin-idp-end-product.md) | Thin IdP end product; PDP remains the risk core | Accepted |
| [0013](0013-idp-staged-start.md) | Start thin IdP in stages; skip Horizon A demo kit | Accepted |
| [0014](0014-thesis-scale-idp-platform.md) | Thesis-scale IdP platform (Authentik/Auth0-shaped, not enterprise) | Accepted (MFA cut line amended by 0022) |
| [0015](0015-rba-is-the-thesis-core.md) | RBA is the thesis core; the IdP is how it is delivered | Accepted |
| [0016](0016-hosted-login-on-idp.md) | Hosted login is served by `rba-idp` (same origin as the PEP API) | Accepted (end-user reasons amended by 0023) |
| [0017](0017-admin-console-colocated-on-idp.md) | Admin console colocated on `rba-idp`; control-plane reads stay with owners | Accepted |
| [0018](0018-live-decision-browser-reads-pdp.md) | Live decision browser reads the PDP `decisions` table | Accepted |
| [0019](0019-groups-grant-app-access.md) | Groups grant app access on the IdP; admin stays `is_admin` | Accepted |
| [0020](0020-local-k8s-k3d-helm.md) | Local Kubernetes via k3d + Helm in `rba-infra` | Accepted |
| [0021](0021-prometheus-grafana-in-chart.md) | Prometheus + Grafana in the `rba` Helm chart | Accepted |
| [0022](0022-product-demo-over-gitops.md) | Product demo over further GitOps; real step-up + travel rule | Accepted (Demo-3 UX 0023; colocated `/app` 0024; Demo-2…3 order 0026) |
| [0023](0023-end-user-login-is-opaque.md) | End-user surfaces stay opaque; explainability lives in admin | Accepted (app location amended by 0024) |
| [0024](0024-separate-demo-app.md) | Demo client is a separate app that uses the RBA platform | Accepted (namespace 0025; stage order 0026) |
| [0025](0025-demo-app-separate-namespace.md) | Demo app lives in another namespace and authenticates via the IdP | Accepted |
| [0026](0026-restage-demo-around-tenant-app.md) | Restage Demo-1…4 around a tenant app in another namespace | Accepted (walkthrough scope superseded by 0035) |
| [0027](0027-supervised-second-opinion-and-failed-logins.md) | Supervised second opinion, failed-login bands, success-only familiarity | Accepted (refines 0004; generalises the 0022 escalate) |
| [0028](0028-degrade-to-step-up-and-monitor-mode.md) | PEP degrades to a step-up when the PDP is silent; monitor-only mode in the policy bundle | Accepted (RF-09/RF-10/RNF-03/RNF-08; supersedes the "fail closed → 503" invariant; §2–3 superseded by 0034) |
| [0029](0029-rbac-roles-and-permissions.md) | RBAC: `user → group → role → permission`, app-scoped; `is_admin` stops being an authorization input | Accepted (RF-13/RF-21; supersedes the `access` grant of 0019) |
| [0030](0030-two-themes-one-palette.md) | One token file, two themes: light for end users, dark for operators; severity escalates green → cyan → indigo → plum, never red | Accepted (shared palette across apps amended by 0036) |
| [0031](0031-aws-eks-managed-data-plane.md) | AWS: EKS (Graviton, private subnets), RDS + ElastiCache Valkey, RabbitMQ in-cluster, CloudFront for HTTPS, two Terraform stacks | Accepted |
| [0032](0032-real-login-signals.md) | Real login signals: client IP from trusted proxies, DB-IP Lite geolocation (IPv4 + IPv6), override only for simulated travel | Accepted |
| [0033](0033-multi-tenancy.md) | Multi-tenancy: organisations with isolated directories, policies and decisions; tenant vs. platform operators | Accepted (changes E90 scope) |
| [0034](0034-base-policy-and-engine-mode.md) | Base policy (password-only / always-MFA) + engine mode (off / monitor / active); monitor keeps the existing control | Accepted (supersedes 0028 §2–3; closes RF-08) |
| [0035](0035-demo-around-personas-and-tenants.md) | Final demo: Martín and Diego, Roku Bank and Kaze Labs, attacker Lambdas, presenter panel as stage machinery | Accepted (supersedes 0026 walkthrough scope) |
| [0036](0036-brand-and-per-tenant-login.md) | RokuSec brand, login branded per tenant, customer apps with their own identity, Spanish UI | Accepted (amends 0030) |
| [0037](0037-passkey-enrollment-from-trusted-context.md) | A passkey is enrolled only after a low-risk login or with an operator-issued activation code; a step-up without a passkey is denied | Accepted (amends Demo-4 / 0022) |
| [0038](0038-console-on-carbon.md) | The console is built on IBM's Carbon (g100) with the logo's blues, IBM Plex self-hosted, Carbon's UI shell | Accepted (amends 0030, 0036 for the console) |
| [0039](0039-public-client-ids.md) | Login URLs name applications by an opaque public client ID (`rk_…`); the internal application ID never reaches the browser | Accepted |
| [0040](0040-scores-step-up-rules-block.md) | A risk score steps up and never blocks; only rules block. Default bands 0.40 / 0.60 / 0.80 (from D-2) | Accepted (amends 0034) |
| [0041](0041-stage-machinery-wiring.md) | Attackers via function URLs + shared secret; presenter panel inside the bank; load hits the PDP; reset touches only profiles; no TEST-NET seed in the cloud | Accepted |
| [0042](0042-learn-from-completed-logins-and-deployment-population.md) | Profile learns only completed logins (`/risk/confirm` after a passed step-up); browser/OS scored as families; Freeman population learned by the deployment (κ = 100); freeman-0.3.0, logreg-0.2.0 | Accepted |
| [0043](0043-hybrid-freeman.md) | freeman-0.4.0: network as Freeman's hierarchy rooted at the population, flat device/OS/browser; hour out of the score; profile rebuild | Accepted |
| [0044](0044-decision-path-in-valkey.md) | Decision path touches Valkey only: journal key + stream; publisher fills the Postgres read model; policies refreshed in the background | Accepted |
| [0045](0045-block-the-source-not-the-account.md) | Repeated wrong passwords: block the source IP, step the account up after an attack, pause it only for the owner's own failures | Accepted |

## Template

```markdown
# ADR-NNNN: <title>

- Status: Proposed | Accepted | Superseded by ADR-XXXX
- Date: YYYY-MM-DD

## Context
<what forces a decision; constraints; options considered>

## Decision
<what we decided>

## Consequences
<positive + negative results of the decision; follow-ups>
```
