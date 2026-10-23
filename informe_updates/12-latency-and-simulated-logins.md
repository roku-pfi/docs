# RNF-07: measurement, decision path, simulated logins (2026-10-23, first cloud e2e)

- **Where p95 is measured:** at the PDP, from its own histogram
  (`http_request_duration_seconds`, `/risk/evaluate`, Prometheus), over the
  presenter's 90 s load (32 concurrent clients). The presenter's own figure
  also counts its queueing on a small pod (1.9 s vs. 336 ms at the PDP on the
  first run), so it is reported but not judged. A 0.2 s bucket makes the 200 ms
  bound an exact edge.
- **Cloud sizing for it:** decision-service 3 replicas × 400m CPU (limit 1),
  HPA up to 5 on the 2 × t4g.medium nodes. The first run (2 × 50m, limit 500m)
  scaled late and throttled.
- **The decision path no longer touches Postgres** (ADR-0044): profile,
  population, policy cache and a Valkey "decision journal"; the event-publisher
  fills the decisions table behind it. Cloud result: **p95 23–24 ms** at ~165
  decisions/s, 0 errors (from 0.5–2.5 s while the save was in the path; run
  table in finding 2026-10-23-cloud-decision-latency). Limitation to state:
  Valkey is a single node without persistence, so losing it loses the
  decisions not yet copied (normally under a second's worth).
- **Simulated logins teach nothing:** a login with the presenter's country/ASN
  override (ADR-0032) is scored and shown as *simulado*, but never updates the
  profile or the population prior. Before, the simulated trip to Japan made
  Japan familiar and the Tokyo attack after it scored MEDIUM.
- **Roku Bank's CRITICAL band steps up instead of blocking** (policy 2.1.0,
  as D-2 already set for new tenants). With the fix above, the Tokyo attacker
  (new country, network, OS, hosting ASN) scores CRITICAL (0.88); a passkey
  stops it as well as a block, and a real customer abroad can still get in.
  Blocks come only from the failed-login rules.
- **Review:** non-functional requirements / results (how RNF-07 is measured and
  the cloud numbers), architecture (the decision path and the async plane:
  journal → publisher → read model + RabbitMQ, replacing the outbox),
  limitations (Valkey durability), deployment sizing, Roku Bank's policy table,
  and the learning rule next to ADR-0042's "only completed logins".
