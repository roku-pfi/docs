# ADR-0023: End-user surfaces stay opaque; explainability lives in admin

- Status: Accepted
- Date: 2026-08-16
- Amends [ADR-0022](0022-product-demo-over-gitops.md) Demo-3 (“show action +
  reasons on the banking home”) and the end-user reading of
  [ADR-0016](0016-hosted-login-on-idp.md) (“reasons are first-class UI” on
  hosted login). Admin / operator surfaces are unchanged
  ([ADR-0015](0015-rba-is-the-thesis-core.md),
  [ADR-0018](0018-live-decision-browser-reads-pdp.md)).

## Context

ADR-0022 put this session’s action and top reasons on the banking `/app` home
so the defense could see *why* without opening admin. That turns the client
into a risk dashboard. Real banking apps do not show scores, LLRs, or
`ALLOW` / `REQUIRE_MFA` to the customer. The product claim is **adaptive
friction that feels like a normal login**, with explainability for the
operator (and the thesis), not for the account holder.

Options considered:

- Keep reasons on `/app` (rejected).
- Show reasons only on the IdP challenge, hide them in the app (still leaks
  internals to the person logging in).
- **Opaque end-user path; reasons only in admin** (chosen).

## Decision

1. **The banking UI is a normal app.** After `AUTHENTICATED`, `/app` shows
   balances / a dummy home — no `risk_score`, `action`, `risk_level`, or
   reason list.

2. **Hosted login is also opaque to the end user.**
   - `ALLOW` → redirect to `/app` (no “why this decision” success panel).
   - `REQUIRE_MFA` / `REAUTHENTICATE` → generic step-up (“confirm it’s you”);
     not a contribution table.
   - `BLOCK` → generic rejection, not a score.

3. **Explainability stays on operator surfaces:** admin Decisions (and
   policy). The walkthrough uses a **second window on admin**, not the
   customer home. Demo-2 scenario control is presenter-only, not customer
   chrome.

4. The PDP still returns reasons on `/risk/evaluate`. The IdP still stores
   them on the challenge/decision. They are just not rendered to the
   account holder.

## Consequences

- Positive: the demo matches how RBA is supposed to feel (friction only when
  needed, no lecture). Admin remains the thesis differentiator.
- Negative: a one-laptop demo needs admin open beside the app, or the
  audience will not see reasons unless the presenter switches windows.
- Follow-up: Demo-3 copies this; do not add a “debug reasons” toggle on `/app`
  as a default. A local-only presenter flag is allowed later if the two-window
  flow is painful — it must stay off for the customer path.
