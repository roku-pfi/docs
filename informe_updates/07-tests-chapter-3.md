# Tests for chapter 3 (pruebas)

- **New:** end-to-end Playwright suite of the scenario map (`rba-infra/e2e`,
  `make e2e`), one test per scene, with a virtual passkey. Green 3 runs in a
  row on k3d (`docs/findings/2026-10-22-e2e-scenario-map.md`). Load run: ~10k
  PDP requests, HPA 2 → 5 replicas, p95 ~800 ms on a laptop (not an RNF-07
  number).
- **Found by the tests:** the outbox relay was capped at ~50 events/s, delaying
  failed logins by minutes after load; fixed. Worth a line as a test that found
  a real defect.
- **Pending:** AWS run + p95 against RNF-07 (200 ms) — add once D-3 is done.
