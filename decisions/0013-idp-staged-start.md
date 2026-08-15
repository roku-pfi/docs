# ADR-0013: Start the thin IdP in stages; skip the Horizon A demo kit

- Status: Accepted
- Date: 2026-08-15
- Amends sequencing in [ADR-0012](0012-thin-idp-end-product.md) §2 (architecture
  unchanged: IdP = PEP, decision-service = PDP)

## Context

ADR-0012 kept a ~10-day **PDP demo** (Horizon A) before coding `rba-idp`. A
scripted walkthrough was started and then discarded — it is not the product.

The end product is still the thin IdP wrapping the existing risk plane. Building
login + MFA + session + UI + admin in one slice would recreate the “do everything
now” failure mode.

## Decision

1. **Skip** Horizon A demo-kit polish (no presenter scripts, no throwaway PEP UI).
2. **Start Phase 7 now**, one stage at a time. Do not stand up admin, frontend, or
   groups until the listed predecessors are done.
3. **Stages** (canonical copy: `plans/development_plan.md` §8 Phase 7):
   - **IdP-1** Contracts — freeze login / MFA / session / public-user shapes.
   - **IdP-2** Identity store — `rba-idp` + `rba_idp` DB; password verify only.
   - **IdP-3** PDP enforce — call `/risk/evaluate`; map action → login outcome.
   - **IdP-4** Session + mock MFA — cookie/token on ALLOW; challenge + OTP.
   - **IdP-5** Login UI — pages that call the IdP API.
   - **IdP-6** Admin — users, decisions, policy.
   - **IdP-7** Stretch — groups / app-scoped permissions.
4. Identity never moves into `decision-service`. PDP contracts stay v0.1.x.

## Consequences

- Positive: work is additive toward October; each stage is demoable; PDP latency
  boundary stays intact.
- Negative: no curl-script “wow” path until IdP-3/4.
- Follow-up: IdP-1 lands as `rba-contracts` 0.2.0 (new API, PDP unchanged).
