# Final phase: from a laptop prototype to a product running on AWS

Plan agreed on 2026-10-21. Checkboxes are mirrored in [`status.md`](status.md);
the reasoning lives in ADRs 0031–0036.

## Goal

Make RokuSec look and behave like an end product, running in the cloud, and
demonstrate it doing its job for the two people it is for: **Martín**, a customer
who only experiences the login, and **Diego**, the operator who explains and
tunes it. Two customer organisations share the platform: **Roku Bank** and
**Kaze Labs**.

## Decisions

| ADR | Decision |
|---|---|
| [0031](../decisions/0031-aws-eks-managed-data-plane.md) | EKS (Graviton, private subnets + NAT) · RDS PostgreSQL · ElastiCache Valkey · RabbitMQ in-cluster · CloudFront for HTTPS · two Terraform stacks · ECR + GitHub Actions OIDC · secrets from Terraform |
| [0032](../decisions/0032-real-login-signals.md) | Client IP from trusted proxy headers · DB-IP Lite (country + ASN, IPv4 + IPv6) in the image · override kept only for simulated travel |
| [0033](../decisions/0033-multi-tenancy.md) | Tenants with isolated users, roles, policy, decisions · login finds the tenant from the application · tenant operators vs. platform operator |
| [0034](../decisions/0034-base-policy-and-engine-mode.md) | `base_policy` (password-only / always-MFA) + `engine_mode` (off / monitor / active) per tenant · monitor enforces the base policy |
| [0035](../decisions/0035-demo-around-personas-and-tenants.md) | Demo around Martín and Diego · Roku Bank + Kaze Labs · attacker Lambdas in three regions · presenter panel as stage machinery · console as the live view |
| [0037](../decisions/0037-passkey-enrollment-from-trusted-context.md) | Passkeys enrolled only after a low-risk login or with an operator-issued activation code; no passkey → step-up denied |
| [0036](../decisions/0036-brand-and-per-tenant-login.md) | RokuSec brand · login branded per tenant (`rokusec` / `tenant` + "Protegido por RokuSec") · apps with their own identity · Spanish UI, English logs |

## Workstreams and stages

Stages are numbered per workstream. Dependencies are listed where they exist;
everything else can run in parallel.

### B: Brand

- [x] **B-1** Logo as vector: primary, horizontal, inverse, mono, favicon
      (`design/brand/build.py` → `docs/brand/`).
- [x] **B-2** RokuSec tokens move to the logo blues; favicon and logo wired into
      the IdP and console.
- [ ] **B-3** Roku Bank identity (name, logo, palette, type).
- [ ] **B-4** Kaze Labs identity.
- [ ] **B-5** Brand rationale + short guide for the informe (Spanish).

### T: Tenancy and policy (the core code change)

- [x] **T-1** Contracts (`rba-contracts` 0.9.0): `tenant_id` on evaluate request / decision / event;
      per-tenant policy endpoints; `base_policy`, `engine_mode`,
      `enforced_action`; remove `monitor_only` / `monitored_action`.
- [x] **T-2** IdP: `tenants` table; `tenant_id` on users, applications, groups,
      roles; email unique per tenant; login resolves the tenant from `client_id`;
      tenant branding fields; operator email domains (console login by domain,
      uniform errors); `rokusec` system tenant.
- [x] **T-3** Done 2026-10-21 (PDP side, then IdP enforcement with T-2). PDP: one policy bundle per tenant, **stored in Postgres** (today
      `PUT /policy` only updates the replica that served it — fixed here); `engine_mode` and
      `enforced_action`; decision reads filtered by tenant. IdP enforces
      `enforced_action`, skips the PDP when `OFF`. *(after T-1)*
- [x] **T-4** Profile and audit services carry `tenant_id` through (audit reads
      scoped by tenant). *(after T-1)*
- [x] **T-5** Console authorization: tenant operators scoped server-side;
      platform-operator role; a cross-tenant test per admin endpoint. *(after T-2)*
- [x] **T-7** Passkey enrollment only from a trusted context (ADR-0037):
      post-ALLOW offer, operator activation codes, step-up without a passkey
      denied. *(after T-2)*
- [x] **T-6** Seed: Roku Bank and Kaze Labs with users, groups, roles,
      operators and starting policies; one platform operator. *(after T-2, T-3)*

### C: Cloud-readiness in code

- [x] **C-1** Configuration from the environment: secrets, WebAuthn RP ID and
      origins, cookie `Secure`, public URLs of each surface. Everything was
      already env-driven; `DEPLOYMENT=cloud` now refuses local-only defaults
      (IdP, bank) and the bank cookie is `Secure` there.
- [x] **C-2** Client IP from trusted proxy headers (`TRUSTED_PROXY_CIDRS`).
      Also: the login body's `ip_address` is no longer believed in the cloud,
      private addresses stop scoring as home there, IPv6 is routable.
- [x] **C-3** IP geolocation with DB-IP Lite, IPv4 + IPv6; `simulated` flag on
      decisions taken with an override. In the cloud an override needs a signed
      simulation ticket (`POST /simulation/ticket`, presenter secret).
- [x] **C-4** Schema creation and seed as Kubernetes Jobs (not on pod start,
      so replicas do not race).
- [x] **C-5** Multi-arch images (arm64 + amd64) for every service.
      `rba-infra/scripts/build-images.sh` (buildx); all six build for both, the
      amd64 IdP verified running under emulation.

### I: Infrastructure on AWS

- [x] **I-1** OpenTofu `bootstrap` + `foundation` (`rba-infra/tofu/`): S3 state,
      VPC (2 AZ, public + private, S3 endpoint), ECR ×7, CloudFront ×3, GitHub OIDC
      role, AWS Budgets alert. Applied 2026-10-21. Hosts: auth
      `d1fg623tjiucw7.cloudfront.net`, bank `d3m6kfm1gma0g6.cloudfront.net`, kaze
      `d14wc5kuicqxf6.cloudfront.net`.
- [x] **I-2** Applied 2026-10-22 by `make aws-up` (67 resources). Terraform `runtime`: NAT, EKS + Graviton node group, RDS,
      ElastiCache Valkey, AWS Load Balancer Controller, metrics-server.
- [x] **I-3** `values-aws.yaml` for both charts; Ingress by CloudFront host;
      Secrets from Terraform. *(after I-2, C-1)*
- [x] **I-4** `make aws-up` / `make aws-down` verified end to end 2026-10-22, including re-runs and a real phone. `make up` / `make down`, including re-pointing CloudFront to the
      new ALB; first end-to-end login over HTTPS with a passkey.
- [ ] **I-5** GitHub Actions: reusable test → build → push workflow in every
      service repo; deploy step when a cluster is up.
- [ ] **I-6** Attacker Lambdas in `ap-northeast-1`, `eu-central-1`, `sa-east-1`.

### U: Interfaces

- [x] **U-1** Console shell: logo, tenant name / switcher, navigation, account
      menu, es-AR formatting, reason codes translated.
- [ ] **U-2** Console pages on Carbon (ADR-0038). Usuarios approved; Decisiones done
      (expandable why, own page per decision, filters, auto-refresh). Rest legacy. Console pages: Resumen, Decisiones (live, filters, detail page),
      Política (base policy, mode, bands, saved-prompt figure), Usuarios, Grupos,
      Roles, Aplicaciones, Organizaciones (platform operator). *(after T-3, T-5)*
- [ ] **U-3** Hosted login in both styles (`rokusec`, `tenant`), plus step-up and
      blocked pages. *(after T-2)*
- [ ] **U-4** Roku Bank rebuilt on its own identity. *(after B-3)*
- [ ] **U-5** Kaze Labs app (`rba-demo-kaze`): portal home, a page only
      `admin` can reach, a page `viewer` cannot change. *(after B-4, T-6)*

### D: Demos

- [ ] **D-1** Presenter panel: attacker scenarios (Lambdas), simulated travel,
      load test, reset demo; shared-secret protected. *(after I-6)*
- [ ] **D-2** Policy comparison: offline replay of the held-out split under
      password-only / always-MFA / adaptive, with confidence intervals →
      finding + JSON → the console's Efectividad page.
- [ ] **D-3** Rehearse the scenario map (ADR-0035) end to end on AWS.
- [ ] **D-4** Rewrite `defense-runbook.html` around the new script.

### Found on the first cloud session (2026-10-22)

- [ ] Don't seed the fake TEST-NET home history in the cloud: it never matches a
      real login, so Martín's first real phone login scored CRITICAL (verdict
      BLOCK, enforced passkey in MONITOR). Let real logins build the profile.
- [ ] Onboarding UX: activation inside the login flow and an activation link
      from the console (discussed; current standalone `/activate` accepted for now).

- [x] Local in-memory IdP (`USE_MEMORY_DB`) shared one SQLite connection across
      threads; sessions now take turns there.

### Parked (come back after the main work)

- [ ] Alarms on risk events (RF-16).
- [ ] IdP metrics and MFA outcome in the record (RF-11 / RF-18).
- [ ] Stable device identifier (RF-02 device part). Until then, "new device"
      relies on browser / OS / ASN familiarity.
- [ ] Progressive delay on repeated failures (RF-07).
- [ ] Real impossible travel from abroad, as an asynchronous demo.
- [ ] Per-tenant model; custom domains per tenant.

### After the system is complete (informe chapter 3: pruebas)

- [ ] End-to-end tests of the scenario map (Playwright), also in CI.
- [ ] Load test against AWS; p95 against RNF-07.
- [ ] Usability sessions with real users on the cloud deployment.

## Order of work

```
B-1 ✓
C-1..C-5 ─┬─> I-3 ─> I-4 ─> D-1 ─> D-3 ─> D-4
I-1 ─> I-2┘          │
T-1 ─> T-2 ─> T-3 ─> T-5 ─> T-6 ─> U-2, U-3, U-5
B-2..B-4 ───────────────────────> U-1, U-4, U-5
D-2 (independent, offline)
```

1. **Tenancy and policy first** (T-1 … T-6): the largest change, and every
   screen depends on its shape. Done locally on k3d.
2. **Cloud in parallel** (C, then I): code changes are small; Terraform can be
   written while T is in progress. First goal: today's system on AWS over HTTPS
   with a working passkey.
3. **Interfaces** (U) once the tenancy model is stable, with the brand work (B)
   feeding them.
4. **Demos** (D) on the deployed system; D-2 can be done any time.
5. Parked gaps, then tests and their results for the informe.

## Document changes this implies (E90)

- §1.2.2: remove "Arquitectura multi-tenant" from the exclusions.
- §3.2: RF-08 and RF-09 descriptions to match ADR-0034; a requirement for
  tenant isolation.
- §3.3: screens regenerated; per-tenant login branding.
- §3.4.2 and the architecture figure: AWS deployment (ADR-0031).
- §3.5: AWS services in Tecnologías (EKS, RDS, ElastiCache/Valkey, CloudFront,
  Terraform, GitHub Actions).
- Branding section: logo rationale (`docs/brand/README.md` as a starting point).
