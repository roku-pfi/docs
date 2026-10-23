# Cloud decision latency under load (2026-10-23)

RNF-07: p95 of `POST /risk/evaluate` < 200 ms. Load: the presenter's scene,
32 concurrent clients for 90 s against the PDP Service, on EKS (2 × t4g.medium),
RDS db.t4g.micro, ElastiCache Valkey cache.t4g.micro. p95 read from the PDP's
own histogram (Prometheus); the presenter's figure includes its own queueing.

| Run | Change under test | Requests | Errors | PDP p95 |
|---|---|---|---|---|
| 1 | baseline: 2 × (50m req / 500m limit) | 4,589 | 0 | 336 ms (presenter: 1,892) |
| 2 | 3 × (400m / 1 CPU) | 4,684 | 0 | 99 ms (presenter-bound, no scaling) |
| 3 | presenter pod 250m → 1 CPU | 9,550 | 17 | 181 ms |
| 4–6 | same, outbox growing | 3,500–4,850 | 78–157 | 1.4–2.5 s |
| 7 | partial index on unpublished outbox rows | 3,827 | 108 | 2.5 s |
| 8 | per-step histogram added | 4,699 | 78 | 2.5 s |
| 9 | DB pool: 10 kept connections, no overflow | 7,921 | 0 | 821 ms |
| 10 | one transaction per decision | 9,256 | 0 | 487 ms |
| 11 | `synchronous_commit=off` on the save | 9,097 | 10 | 456 ms (reverted) |
| 12 | **ADR-0044: no Postgres on the path** | 14,925 | 0 | **24 ms** |
| 13 | same, full suite | 17,225 | 0 | **23 ms** |
| 14 | + read-model indexes; full suite 12/12 | 17,298 | 0 | **24 ms** (3 → 5 replicas) |

Per-step p95 before ADR-0044 (run 10): profile read 5 ms, population read
5 ms, scoring 1 ms, policy 1 ms, **save 459 ms**. The save's median was 18 ms:
the tail was waiting on the shared, burstable database (three consumers write
to the same instance), not work the PDP did. After ADR-0044 the slowest step
is the journal write to Valkey.

Throughput rose with it (~50 → ~165 decisions/s): with the same 32 clients,
a shorter answer means more answers.

Reproduce: `make aws-up`, then `E2E_LOAD=1 E2E_P95_MS=200 make e2e` with the
cloud variables (e2e/README.md); per-step numbers from
`rba_evaluate_stage_seconds` in Prometheus.
