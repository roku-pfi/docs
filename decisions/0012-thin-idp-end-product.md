# ADR-0012: Thin IdP end product; PDP remains the risk core

- Status: Accepted
- Date: 2026-08-12

## Context

Phases 1–4 built an explainable **PDP** (`POST /risk/evaluate`) plus async
profile/audit. That matches the original plan’s PEP/PDP split: a caller enforces;
RBA only decides risk actions.

Product intent has widened. For a **near-term demo (~10 days)** the PDP thin slice
is enough. For a **late-October end product**, the system should present as a
**thin IdP**: demo client apps hand login to RBA, which verifies identity lightly,
asks the PDP for risk, and enforces ALLOW / MFA / REAUTH / BLOCK. An admin surface
should look IAM-adjacent (users, optionally groups/permissions, decisions,
policy) — not only a raw risk API.

Risk: over-building the near-term demo in a shape that must be thrown away for the
IdP, or conversely stuffing identity into `decision-service` and breaking the
latency / PDP boundary.

Options considered:
- Stay PDP-only forever (admin + demo PEP UI only).
- Replace the PDP with a monolithic login/IAM app.
- **Add a thin IdP as PEP** that owns identity + session; keep decision-service as
  PDP (chosen).

## Decision

1. **End product (late October):** thin IdP + admin console wrapping the existing
   risk plane. New service(s): `rba-idp` (login, local user store, session, MFA
   challenge) and expanded `rba-admin-api` / `rba-frontend` (users, decisions,
   policy; **groups/permissions as a valid stretch** — app-scoped roles for demo
   clients, not enterprise directory/SCIM/OIDC federation).
2. **Near-term demo (~10 days):** keep shipping the **current PDP + async plane**.
   Do not start IdP implementation until after that demo unless a one-page curl/
   scripted walkthrough is insufficient.
3. **Migration / non-throwaway rule:** near-term work must stay **additive**:
   - **Keep investing in:** `rba-features`, `rba-contracts`, `decision-service`,
     outbox → RabbitMQ → profile/audit, policy config, Freeman artifact, infra
     data plane. These are the IdP’s risk backend forever.
   - **Do not commit near-term to:** identity/password logic inside
     `decision-service`; a “final” admin UX that assumes there is no IdP; OIDC/
     SAML/SCIM; treating the demo PEP as the product.
   - **IdP is a new PEP boundary**, not a rewrite of the PDP. Login flow becomes:
     credentials → IdP → `/risk/evaluate` → enforce action → session.
4. **Groups/permissions:** in-product fit for the October admin story (who may
   access which demo app / admin capability). Explicitly **stretch / cut-last** —
   ship users + decisions + policy first; add groups/roles if calendar allows.
5. **Non-goals (still):** full IAM replacement (federation, org directory sync,
   fine-grained authorization engines). Thesis grade remains explainable RBA +
   microservices/k8s; the IdP is the product shell.

## Consequences

- Positive: demo path unchanged; October product matches “owns client login”
  without discarding Phases 1–4; PDP/PEP stays clean for grading and latency.
- Negative: extra service(s) and UI work in Phases 7+; groups/permissions can
  tempt scope creep — cut order must list them after core IdP login.
- Follow-up: update `development_plan.md` (canonical roadmap) and `status.md`;
  freeze IdP/admin contracts in `rba-contracts` when Phase 7 starts.
