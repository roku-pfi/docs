# ADR-0026: Restage Demo-1…4 around a tenant app in another namespace

- Status: Accepted
- Date: 2026-08-16
- Amends Horizon D sequencing in [ADR-0022](0022-product-demo-over-gitops.md)
  (Demo-2 was seed + scenario picker **before** a client app). Builds on
  [ADR-0024](0024-separate-demo-app.md) /
  [ADR-0025](0025-demo-app-separate-namespace.md).
  Demo-1 **code** is unchanged (travel rule still stands).

## Context

Demo-1…4 were written for a walkthrough **on the IdP**: hosted login, then a
mocked `/app`, with seed and a scenario picker so you could fake country/VPN
without a real client. The product is now a **tenant app in namespace `demo`**
that authenticates through the IdP.

Old Demo-2 (scenario dropdown before any bank exists) and old Demo-3 (the app)
are in the wrong order for that story. Seed still matters — without a usual
home profile every login looks new.

Options considered:

- Keep seed + scenarios as Demo-2, app as Demo-3 (rejected — scenarios have
  nowhere to live except IdP query params).
- Skip seed and only build the app (rejected — home ALLOW never shows).
- **App + seed next; scenarios after the app exists** (chosen).

## Decision

| Stage | Role now | Status |
|---|---|---|
| **Demo-1** | **Platform RBA:** country on the login path; centroid travel escalate; VPN skip. Not a tenant UI. Query `?country=&asn=` stays a low-level override. | **Done** |
| **Demo-2** | **Tenant uses the platform:** `rba-demo-banking` in ns `demo`; browser → IdP; opaque home; seed `demo@example.com` home profile so a usual login can ALLOW. | Next |
| **Demo-3** | **Walkthrough controls:** presenter picks the next login context (home / new country / teleport / VPN). The **app** (or a presenter-only redirect) forwards that context to the IdP. Not a dropdown on the customer home (ADR-0023). | After Demo-2 |
| **Demo-4** | WebAuthn for `REQUIRE_MFA`. Unchanged. | Last |

Walkthrough definition of done is unchanged: home ALLOW → bank; novel country
→ MFA (Freeman); teleport → MFA (rule); VPN → MFA as untrusted network;
**admin Decisions** shows why.

## Consequences

- Positive: Demo-2 is the picture you want (another namespace authenticates
  via your service). Demo-1 stays the reason teleport MFA works. Scenarios
  are chrome on a real relying party, not a fake IdP demo kit.
- Negative: Demo-2 is larger (repo + Helm + `redirect_uri` + seed).
- Follow-up: implement Demo-2 next. Do not skip it to chase passkeys.
