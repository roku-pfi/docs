# End-to-end tests of the scenario map (k3d)

- Date: 2026-10-22
- Suite: `rba-infra/e2e` (`make e2e`, `E2E_LOAD=1 make e2e`)
- Environment: k3d on the development laptop, all services from `main`;
  attackers are the local stand-ins (no Lambdas locally, ADR-0041).

Real Chromium through the hosted login, passkeys on Chrome's virtual
authenticator enrolled with activation codes; Diego's view read from the console
API with his own session. One test per row of the scenario map (ADR-0035).

## Result

| Scene | Martín / attacker sees | Diego sees (asserted) | Result |
|---|---|---|---|
| Monitor mode | passkey, gets in | `MONITOR`, enforced `REQUIRE_MFA`, engine `ALLOW` | pass |
| Switch to active | straight in | `ACTIVE`, `ALLOW`/`ALLOW`, `LOW` | pass |
| Impossible travel | passkey, gets in | `simulated`, `REQUIRE_MFA`, `impossible_travel` | pass |
| Login from Tokyo | attacker stops at `MFA_REQUIRED` | challenge enforced, country `JP` | pass |
| Stuffing burst (3) | passkey again | `REAUTHENTICATE`, `failed_login_burst` | pass |
| Stuffing lockout (12) | "blocked" panel, no challenge | `BLOCK`, `failed_login_lockout` | pass |
| Engine down | passkey, gets in | (not recorded: the store is behind the PDP) | pass |
| Kaze admin | all three pages | — | pass |
| Kaze viewer | read-only; POST 403; settings 403 | — | pass |
| Isolation | — | no Kaze user or decision in Roku Bank's console | pass |
| Load | — | 10,162 requests, 0 errors, PDP 2 → 5 replicas, p95 814 ms* | pass |

10 scenes in about 1 minute, 11 with load in about 4. Three consecutive runs
green after the fixes below (the first one with load).

\* The p95 is a laptop figure with five PDP replicas sharing its CPU, not an
RNF-07 measurement; the cloud run asserts `E2E_P95_MS=200`.

## What the tests found

1. **Profiles lagged minutes behind after a load.** The event publisher slept
   its poll interval (1 s) after every batch of 50, capping the relay at about
   50 events per second; the load scene writes about 115. After it, a stuffing
   burst reached Martín's failure window about four minutes late, so his next
   login was let in. Probe, a burst every ~20 s after a 90 s load:

   | | failures in the window 8 s after each burst |
   |---|---|
   | before | 0 for ~5 min after the load, then 36 at once (every burst together) |
   | after  | 3 from the first burst after the load ends |

   Fixed in `rba-event-publisher`: a full, healthy batch is followed at once by
   the next. During the load itself the profile consumer is still the
   bottleneck, so stuffing should not be run while the load scene is on.
2. **A policy change takes up to 5 s to apply.** The IdP drops its cached
   policy on save, but each PDP replica keeps its own for `policy_cache_seconds`
   (5). A login within those seconds of "switch to active" was still decided in
   monitor mode. Left as is (RNF-05 asks for no redeploy, not for zero delay);
   the suite waits it out and the run sheet should leave a breath between
   Diego's save and Martín's login.
