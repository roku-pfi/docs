# Where the project stands (2026-10-23)

One page to decide what to work on next. Details live in `status.md`,
`final-phase.md`, the ADRs and `informe_updates/`.

## 1. The short version

- **The system is done and proven in the cloud.** All 13 end-to-end scenes pass
  on AWS, and the decision p95 is 24 ms under load (target 200 ms). The demo is
  scripted click by click (`defense-runbook.html`).
- **The informe is where the points are.** In
  `material_sources/PFI - Rubrica final.xlsx` (69.9, "aceptado con cambios de
  contenido"), the software-facing sections score well. Two whole sections are
  close to zero:
  - **Desarrollo funcional:** 0.75 of 15.
  - **Desarrollo de negocio:** 1 of 10.

  That is ~23 points, and none of it needs code.

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

### 3.1 Informe: the big wins

| Rubric item | Score now | What it asks | What we already have | What's missing |
|---|---|---|---|---|
| **Diagramas funcionales** (Desarrollo funcional) | NO CUMPLE | Use cases, sequence/activity diagrams | Scenario map (ADR-0035), architecture diagrams | **UML use-case diagram** (Martín, Diego, Juan, attacker); **sequence diagrams** for login + step-up and for the decision path; an activity diagram of the decision |
| **Herramientas visuales UX** | NO CUMPLE | User flows, wireframes, mockups | 12 mockups, a rebuilt UI | **User-flow diagrams** (customer login with step-up; operator switching to active); fresh screenshots of the rebuilt screens |
| **Metodología de desarrollo** | NO CUMPLE | Name the method and justify it | Phased plan, ADR per decision, devlog, findings | **A section naming it** (iterative-incremental with a phase roadmap and decision records; fits a research project where findings change the plan, e.g. ADR-0042/43/44 came from measurements) |
| **Pruebas funcionales** + **Resultados** | NO CUMPLE ×2 | User tests with valid techniques, and conclusions | e2e 13/13 on AWS, load test, usability protocol (note 15) | **Run the usability sessions** (5–8 people, ~30 min each) and write the results; add the e2e and load results as functional/system tests |
| **Desarrollo funcional** (requisitos) | PARCIAL | Requirements incl. non-functional | E50 requirements + audit (e50 note) | The RNF list stated with how each is verified (RNF-07 → 24 ms, etc.) |
| **Modelo de negocio** | NO CUMPLE | How it becomes sustainable | — | **Business Model Canvas**: B2B SaaS for banks/fintechs/workforce, priced per monthly active user, with a free tier; value proposition = fewer passkey prompts at the same protection |
| **Viabilidad económico-financiera** | NO CUMPLE | ≥2 scenarios, financial tools | AWS cost per hour measured (~US$0.27/h per stack) | **Cost model + revenue in optimistic and pessimistic scenarios**, NPV / payback / break-even |
| **Branding y logotipo** | PARCIAL | Logo, rationale, branding concepts | Logo + full system | **Apply note 14** to the informe |
| Errores idiomáticos, hilo conductor | PARCIAL | Language, flow between sections | — | A full proofreading pass; connect chapters (problem → model → system → tests → results) |
| Producto entregado | PARCIAL | Product meets objectives and scope | Everything in §2 | **Apply notes 01–13**: the informe still describes the August system |

These are mostly writing. Diagrams, the business model and the financial
analysis can be drafted with my help from what exists. Only the usability
sessions need other people's time.

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
| **Stable device identifier** (RF-02) | "Device" is the *class* of device read from the browser (desktop, macOS, Chrome). Every Mac with Chrome looks the same to the model. | A long-lived random device id (cookie set by the IdP) so "known device" means *this* laptop. A stronger signal against attackers who copy the user agent. | **Medium**: closes a listed requirement gap and strengthens the model's weakest signal. | **High**: new feature, schema change, retrain and re-evaluate the model, privacy note. |
| **Activation inside the login flow** | New users and operators enroll their passkey on a separate `/activate` page with a code we generate by command or from the console. | An invite link from the console; the person sets the password and passkey in one flow. | **Low–medium**: onboarding polish. Usability tests might flag it. | **Medium**: IdP flow and console invite. |
| **Real impossible travel** | The demo injects the country (marked *simulado*). A VPN abroad doesn't work: hosting networks switch the travel rule off on purpose. | A real second login from a residential network abroad, recorded as an asynchronous demo. | **Low**: realism only; the rule itself is tested. | Needs someone abroad (or a residential proxy). |
| **Per-tenant model** | One Freeman model for every tenant; each tenant has its own thresholds and policy. | A model trained per tenant on its own logins. | **Low** for the thesis: no tenant has enough real data. | **High**. |
| **Custom domains per tenant** | Roku Bank's branded login runs on RokuSec's `*.cloudfront.net` host. | `login.rokubank.com`: DNS, certificates, a WebAuthn relying party per tenant. | **Low–medium**: completes the white-label illusion. | **Medium–high**, plus a real domain. |
| **Always-MFA per user or app** (RF-08) | "Passkey siempre" exists per tenant (base policy). | A per-user or per-application switch. | **Low–medium**: closes RF-08 fully. | **Low–medium**. |
| **LDAP** (RF-19) | Not built. | Resolve users from an existing directory. | **Low**: recommended to demote to optional in the requirements (e50 note). | High. |
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
