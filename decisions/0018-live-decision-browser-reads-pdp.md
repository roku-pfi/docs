# ADR-0018: Live decision browser reads the PDP `decisions` table

- Status: Accepted
- Date: 2026-08-15
- Amends [ADR-0017](0017-admin-console-colocated-on-idp.md) §3 (audit as the
  admin read model). Audit ownership of `rba_audit` is unchanged.

## Context

IdP-6 admin listed decisions via `rba-audit-service` HTTP. That store only fills
after evaluate → outbox → publisher → RabbitMQ → audit consumer. A successful
login with just IdP + PDP therefore showed an empty Decisions tab — the thesis
differentiator (score + action + reasons) was invisible in the console.

The PDP already persists `decisions` (with reasons) in the same transaction as
`/risk/evaluate`. That row exists before any async hop.

## Decision

1. **`GET /decisions` on `rba-decision-service`** returns the same
   `DecisionListResponse` / `DecisionRecord` contract as the audit read API.
2. **The IdP BFF reads that PDP endpoint by default** (`AUDIT_BASE_URL`
   defaults to `http://localhost:8000`). Admin + login work with two processes.
3. **`rba-audit` remains the async archive.** Point `AUDIT_BASE_URL` at
   `:8002` only when you explicitly want the consumer's copy.

## Consequences

- Positive: every scored login appears in admin immediately; demo no longer
  depends on RabbitMQ for the decision browser.
- Negative: admin list traffic hits the PDP (control plane, not the login
  evaluate path). Acceptable at thesis scale.
- Follow-up: optional later split so a dedicated admin-api reads audit only.
