# 34. Base policy and engine mode: monitor keeps the existing control

- Status: Accepted
- Date: 2026-10-21
- Supersedes: [ADR-0028](0028-degrade-to-step-up-and-monitor-mode.md) §2 and §3
  (monitor-only returns `ALLOW`). ADR-0028 §1 (degrade to a step-up when the PDP
  is silent) stands.
- Related: [ADR-0033](0033-multi-tenancy.md) (the policy is per tenant).
- Requirements: **RF-08** (permanent MFA, still open), **RF-09** (monitor mode),
  **RNF-05**, **RNF-08**.

## Context

ADR-0028 implemented monitor mode as "evaluate, record the verdict, return
`ALLOW`". That lets every user straight in while the engine is being observed,
which is not what an organisation means by monitoring a new control. An
organisation already has a login policy before RokuSec; in a bank, typically
"always ask for the second factor". Monitoring means: **keep doing that, and show
me what the engine would have done instead.** The payments security architect
described the same thing: the engine in monitoring while the organisation
"continues with its own rules".

Separately, RF-08 (permanent MFA as an alternative to adaptive) was never
built, and it is exactly that existing rule.

## Decision

Each tenant's policy bundle gains two fields.

**`base_policy`**: what the organisation enforces without the engine.

| Value | Effect on a login whose password is correct |
|---|---|
| `PASSWORD_ONLY` | allow |
| `ALWAYS_MFA` | require the passkey (RF-08) |

**`engine_mode`**: how the engine's verdict is used.

| Mode | PDP called | Verdict recorded | What is enforced |
|---|---|---|---|
| `OFF` | no | no | `base_policy` |
| `MONITOR` | yes | yes | `base_policy` |
| `ACTIVE` | yes | yes | the engine's verdict |

### Where it is computed

In the PDP, so the policy has one owner and changes need no redeploy (RNF-05).
`RiskEvaluateResponse` carries both:

- `action`: the engine's verdict, always (what Diego analyses);
- `enforced_action`: what the IdP must apply (`action` when `ACTIVE`, the base
  policy's action when `MONITOR`).

The IdP enforces `enforced_action` and nothing else. With `OFF`, the IdP reads
the base policy from the tenant's bundle (cached) and does not call the PDP.

### Interactions

- **PDP unreachable** (ADR-0028 §1, unchanged): the IdP steps up with
  `pdp_unavailable`. With `ALWAYS_MFA` as the base policy that is the same
  outcome the user would have had anyway.
- **Records and metrics** keep the verdict in `action` and add the enforced
  action, so the console can show "applied: MFA (base policy) · engine: allow".
- The decision row and event gain `enforced_action` and `engine_mode` (additive
  contract change).

### The number this makes possible

While a tenant is in `MONITOR` with `ALWAYS_MFA`, every login the engine would
have allowed is a second-factor prompt the engine would have saved. The console
shows that share over the monitoring window, next to how many attempts the
engine would have escalated beyond the base policy. That is the thesis argument,
computed from the tenant's own traffic before the tenant switches the engine on.

## Consequences

- RF-08 closes.
- Monitor mode now means what an operator expects, and the old behaviour remains
  expressible as `MONITOR` + `PASSWORD_ONLY`.
- The demo has a natural arc: Roku Bank starts in `MONITOR` with `ALWAYS_MFA`
  (Martín is asked every time), the operator reads the saved-prompt figure and
  switches to `ACTIVE`, and the next usual login goes straight in.
- `monitor_only` is removed from the contract (replaced by `engine_mode`). ADR-0028's
  `monitored_action` field is removed; `action` / `enforced_action` replace it.
- The ADR-0028 test "the shipped policy enforces" becomes "the shipped tenant
  bundles have the intended mode" (Roku Bank `MONITOR`, Kaze Labs `ACTIVE`).
