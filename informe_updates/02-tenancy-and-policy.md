# Multi-tenancy, base policy, engine mode (ADR-0033, 0034, 0039)

- **Changed:** RokuSec is multi-tenant (Roku Bank, Kaze Labs; logical isolation
  by `tenant_id`, email unique per tenant). Each tenant's policy has a
  `base_policy` (PASSWORD_ONLY / ALWAYS_MFA) and an `engine_mode`
  (MONITOR keeps the base policy and records what the engine would do; ACTIVE
  enforces it). Apps are named by an opaque public `client_id`.
- **Review:** scope (§1.2) if it describes a single organisation; architecture
  and data model; RF-08 (MFA by policy) and any text about "monitor mode".
