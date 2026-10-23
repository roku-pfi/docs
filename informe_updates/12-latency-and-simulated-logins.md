# RNF-07 measurement and simulated logins (2026-10-23, first cloud e2e)

- **Where p95 is measured:** at the PDP, from its own histogram
  (`http_request_duration_seconds`, `/risk/evaluate`, Prometheus), over the
  presenter's 90 s load (32 concurrent clients). The presenter's own figure
  also counts its queueing on a small pod (1.9 s vs. 336 ms at the PDP on the
  first run), so it is reported but not judged. A 0.2 s bucket makes the 200 ms
  bound an exact edge.
- **Cloud sizing for it:** decision-service 3 replicas × 400m CPU (limit 1),
  HPA up to 5 on the 2 × t4g.medium nodes. The first run (2 × 50m, limit 500m)
  scaled late and throttled.
- **Simulated logins teach nothing:** a login with the presenter's country/ASN
  override (ADR-0032) is scored and shown as *simulado*, but never updates the
  profile or the population prior. Before, the simulated trip to Japan made
  Japan familiar and the Tokyo attack after it scored MEDIUM.
- **Review:** non-functional requirements / results (how RNF-07 is measured,
  with the cloud numbers once the rerun passes), deployment sizing, and the
  learning rule next to ADR-0042's "only completed logins".
