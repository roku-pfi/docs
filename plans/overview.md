# Where the project stands (2026-10-23)

One page to decide what to work on next. Details live in `status.md`,
`final-phase.md`, the ADRs and `informe_updates/`.

## 1. The short version

- **The system is done and proven in the cloud.** All 13 end-to-end scenes pass
  on AWS, and the decision p95 is 24 ms under load (target 200 ms). The demo is
  scripted click by click (`defense-runbook.html`).
- **The informe already covers most of the rubric.** The E90 has the business
  model (canvas), the financial analysis with adoption scenarios and
  indicators, the methodology (waterfall, phases applied), wireframes, the login
  flow diagrams, the architecture and the data model. The scores in
  `PFI - Rubrica final.xlsx` predate it.
- **The one missing chapter is testing.** The E90 has no "pruebas" section. The
  rubric asks for user tests and their results; we have the system tests (e2e,
  load), and the user tests are still to run.

## 2. Done

| Area | What exists |
|---|---|
| Model | freeman-0.4.0 (hierarchical network + flat device), supervised second opinion, deployment population, learning only from completed logins (ADR-0042/43). Offline comparison password-only vs. always-MFA vs. adaptive (console Efectividad). |
| Decision engine | PDP with tenant policies, monitor/active modes, escalation rules, decision path on Valkey only (ADR-0044), lockout redesign: block the source, not the account (ADR-0045). |
| Identity provider | Hosted login in three looks, passkeys (WebAuthn), activation codes, enrollment only from a trusted context, RBAC (user → group → role → permission), multi-tenancy, degrade to step-up when the engine is down. |
| Console | Carbon g100: Resumen (server-side summary, alarms), Eventos with reasons and passkey outcome, Efectividad, Usuarios/Grupos/Roles, Aplicaciones, Política, Organizaciones. |
| Customer apps | Roku Bank (own brand, tenant-styled login), Kaze Labs (RokuSec login, roles that matter). |
| Async plane | Valkey journal → event-publisher → read model + RabbitMQ → profile and audit services. |
| Cloud | AWS: EKS 1.35 (Graviton), RDS, Valkey, CloudFront, attacker Lambdas in 3 regions; `make aws-up / aws-down`; passkeys kept across teardowns. |
| Quality | Unit tests in every repo; Playwright e2e 13/13 on AWS; load test with p95 at the PDP; CI on every repo (test → image to ECR). |
| Observability | Prometheus + Grafana, 8 alert rules, IdP and PDP metrics, per-step latency histogram. |
| Brand | RokuSec logo and system, both customer identities, rationale note (14). |
| Docs | 45 ADRs, findings with numbers, devlog, 16 informe notes, demo runbook. |

## 3. What's left, by points

### 3.1 Informe

| Rubric item | What it asks | Status |
|---|---|---|
| **Pruebas funcionales** + **Resultados** | Tests with users, valid techniques, conclusions | **Missing chapter.** We have the system tests (e2e 13/13 on AWS, load p95 24 ms, unit tests in CI); the tests with people (note 15) are still to run. |
| Producto entregado | The product meets objectives and scope | Apply notes 01–16: the E90 describes the August prototype (k3d, outbox, old lockout). |
| Branding | Logo with rationale | Apply note 14. |
| Diagramas funcionales / UX | Use cases, flows, mockups | E90 has wireframes and login flows; a use-case diagram could be added if the tutor wants one. |
| Errores idiomáticos, hilo conductor | Language and flow | A proofreading pass once the new chapters are in. |

### 3.2 System: small and optional

| Item | Value | Effort |
|---|---|---|
| `make aws-up` + e2e once after the demo (validates today's undeployed work: alarms, metrics, summary, passkey outcome) | **High**, it's the proof for those features | 30 min |
| k3d run of the new decision path (planned next week) | Medium: keeps local dev working | 1 h |
| Policy-page impact estimate from the server (like Resumen) | Low–medium | 1 h |
| DLQ / worker metrics; event-lag panel (Phase 4/5 leftovers) | Low for the grade; good engineering | 2–3 h |

## 4. Parked items: what each means and what it's worth

| Item | What it means today | What doing it would change | Value | Effort |
|---|---|---|---|---|
| **Stable device identifier** (RF-02) | "Device" is the *class* of device read from the browser (desktop, macOS, Chrome). Every Mac with Chrome looks the same to the model. | A long-lived random device token (HttpOnly cookie set by the IdP, hashed per tenant): "known device" means *this* laptop. It is learned like the rest of the profile (only from completed logins). It also feeds the ADR-0045 owner-vs-attacker split, which today relies on the network only. | **Medium–high**: RF-02 asks for it explicitly; it is the model's weakest signal today. | **~3 days**: cookie + contracts (½), profile + feature (½), scoring term + rules (1), console (¼), e2e on AWS + ADR + legal note (¾). The dataset has no device ids, so the term cannot be validated offline: justified by design, verified in the e2e. |
| **Activation inside the login flow** | New users and operators enroll their passkey on a separate `/activate` page with a code we generate by command or from the console. | An invite link from the console; the person sets the password and passkey in one flow. | **Low–medium**: onboarding polish. Usability tests might flag it. | **Medium**: IdP flow and console invite. |
| **Real impossible travel** | The demo injects the country (marked *simulado*). A VPN abroad doesn't work: hosting networks switch the travel rule off on purpose. | A real second login from a residential network abroad, recorded as an asynchronous demo. | **Low**: realism only; the rule itself is tested. | Needs someone abroad (or a residential proxy). |
| **Per-tenant model** | One Freeman model for every tenant; each tenant has its own thresholds and policy. | A model trained per tenant on its own logins. | **Low** for the thesis: no tenant has enough real data. | **High**. |
| **Custom domains per tenant** | Roku Bank's branded login runs on RokuSec's `*.cloudfront.net` host. | `login.rokubank.com`: DNS, certificates, a WebAuthn relying party per tenant. | **Low–medium**: completes the white-label illusion. | **Medium–high**, plus a real domain. |
| **Always-MFA per user or app** (RF-08) | "Passkey siempre" exists per tenant (base policy). | A per-user or per-application switch. | **Low–medium**: closes RF-08 fully. | **Low–medium**. |
| **LDAP integration** (E90 §1.2.2, out of scope; asked for in the interviews) | Every tenant uses RokuSec's own user directory. | A tenant can point RokuSec at its existing LDAP / Active Directory. The password is checked against the directory (bind), and the user and their groups are created or updated on each login (just-in-time provisioning, LDAP groups mapped to roles). Passkeys, risk engine and RBAC stay as they are. Kaze Labs (workforce) would be the demo tenant, with an OpenLDAP in the cluster. | **Medium–high**: the interviews' main ask ("a layer next to an existing LDAP"), and it moves an exclusion into scope. | **~3–4 days**: per-tenant directory config + console form (1), LDAP bind + search in the login (½), JIT provisioning + group mapping (½), OpenLDAP in the chart with seed users (½), tests + e2e + ADR (1). |
| **Phase 6: ML lifecycle + data generator** | The model is trained offline by a script; there is no retraining pipeline or synthetic data generator. | Scheduled retraining from the audit store, model registry, drift checks. | **Low–medium**: "future work" material. | **High**. |
| **GitOps** (K8s-3) | Deploys are `make aws-up`. | Argo CD / Flux watching the repos. | **Low**: CI already builds and pushes images. | Medium. |

## 5. Suggested order

1. **Demo** (tomorrow): follow `defense-runbook.html`.
2. **Informe, highest points first:**
   1. functional diagrams, user flows, methodology;
   2. business model and financial analysis;
   3. apply notes 01–16;
   4. proofreading.
3. **Usability sessions** as soon as people are available; they feed two
   rubric items.
4. **System:** one `aws-up` + e2e to validate today's undeployed work; the rest
   only if time remains.
