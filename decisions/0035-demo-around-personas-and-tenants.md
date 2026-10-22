# 35. The final demo: two personas, two tenants, real attackers

- Status: Accepted
- Date: 2026-10-21
- Supersedes: the walkthrough scope of [ADR-0026](0026-restage-demo-around-tenant-app.md)
  (the single-tenant, presenter-driven script). The opacity rule of
  [ADR-0023](0023-end-user-login-is-opaque.md) stands.
- Related: [ADR-0031](0031-aws-eks-managed-data-plane.md),
  [ADR-0032](0032-real-login-signals.md), [ADR-0033](0033-multi-tenancy.md),
  [ADR-0034](0034-base-policy-and-engine-mode.md).

## Context

The current demo is a local walkthrough: a presenter page injects a country and
an ASN, and the scenarios are list items. Everything is simulated, the run
happens on one laptop, and the application behind it is one bank in one tenant.
The final demo has to show the product doing its job, for the people it is for.

## Decision

### Two personas, from E90 §3.1.3

- **Martín**, a customer. He only ever sees an application and its login. He
  never sees a score or a reason (ADR-0023). Light surfaces.
- **Diego**, the operator of one tenant, in the RokuSec console. He sees the
  decisions, why they were taken, latency, policy and the directory. Dark
  surface.
- Plus the **platform operator**, briefly, to show that tenants are isolated.

Scenarios are what make the engine choose different actions. Each one is
triggered by Martín himself, by Diego, or by an attacker.

### Two tenants, two kinds of customer

| | Roku Bank | Kaze Labs |
|---|---|---|
| Kind | consumer banking (B2C) | a software company's internal portal (workforce) |
| Login | **branded as the bank**, "Protegido por RokuSec" | **RokuSec-branded**, as with a workforce IdP |
| Roles | `cliente` | `admin`, `developer`, `viewer` |
| Starting policy | `ALWAYS_MFA` + `MONITOR` | `PASSWORD_ONLY` + `ACTIVE`, stricter bands |
| Shows | the monitor → active rollout | RBAC with roles that matter, a different policy, isolation |

Kaze Labs is a new, small relying party (`rba-demo-kaze`), built like the bank:
it authenticates through the IdP and never calls the PDP.

### Attackers are real processes: Lambdas in other regions

Three small functions in `ap-northeast-1`, `eu-central-1` and `sa-east-1`. Given
a target application, an email and a password list, they run the IdP login flow
from their region. Their egress addresses are real AWS addresses there, so the
IdP sees a real foreign country and a real hosting ASN.

- "Login from Tokyo with Martín's password": MFA as an untrusted network; the
  attacker cannot complete the passkey.
- "Credential stuffing from three continents": failures accumulate,
  REAUTHENTICATE, then BLOCK.

They cost nothing while idle and live in the permanent Terraform stack.

### Impossible travel is simulated and says so

Real impossible travel needs a second login from a residential or mobile network
abroad (ADR-0032). For this demo it stays a presenter override, and the console
marks it **simulado**. A real version is planned later as an asynchronous demo.

### The presenter panel is stage machinery, not product

`/walkthrough` becomes a small presenter panel: one button per attacker scenario,
the simulated travel, the load test, and "reset demo" (re-seed profiles, clear
failure windows). It invokes the Lambdas and Kubernetes jobs, is protected by a
shared secret, and is never shown as part of RokuSec.

### The console is the live view

What the presenter panel used to show belongs to Diego:

- **Decisiones** updates live and has a detail page per attempt (signals,
  contributions, enforced vs. engine action, simulated flag);
- **Resumen**: decisions by action, challenge rate, p95 decision latency against
  the 200 ms target, saved second-factor prompts while in monitor;
- **Efectividad**: the offline comparison of password-only vs. always-MFA vs.
  adaptive on the held-out dataset, with confidence intervals (method recorded
  as a finding when built).

### The scenario map

| Scenario | Triggered by | Martín sees | Diego sees |
|---|---|---|---|
| Monitor mode | Roku Bank's starting policy | passkey every time | applied MFA (base policy) · engine would allow |
| Switch to active | Diego, Política | goes straight in | ALLOW, low risk |
| New device / network | Martín, phone on mobile data | passkey | MFA: unfamiliar ASN / device |
| Login from Tokyo | attacker Lambda | — | MFA: hosting network, JP, challenge not completed |
| Stuffing from three regions | attacker Lambdas | re-enter credentials on his next login | burst → REAUTHENTICATE → BLOCK |
| Impossible travel | presenter panel | passkey | MFA: impossible travel, marked simulado |
| Engine down | Diego scales the PDP to 0 | passkey, never a lockout | MFA: pdp_unavailable |
| Load | presenter panel | — | p95 stays under target, PDP replicas scale |
| Roles | Kaze Labs users | a developer cannot reach admin pages | role assignments in the directory |
| Isolation | platform operator / Roku Bank's Diego | — | Roku Bank's Diego sees nothing of Kaze Labs; the same email exists in both |

The bank's `viewer` user leaves the main script.

## Consequences

- Every scenario except impossible travel is produced by real traffic from real
  places.
- One more deployable (`rba-demo-kaze`) and a repo in the org.
- The defense runbook (`docs/plans/defense-runbook.html`) is rewritten around
  this map once the cloud deployment exists.
- The functional and usability tests that chapter 3 still needs run against
  this same environment.
