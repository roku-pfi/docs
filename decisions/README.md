# Architecture Decision Records (ADRs)

Each ADR captures one decision: the context that forced it, what we decided, and
the consequences. ADRs are immutable once **Accepted** — if we change our mind, we
add a new ADR that **supersedes** the old one (and mark the old one accordingly),
rather than editing history. This preserves the reasoning trail for the thesis.

Read [`status.md`](../plans/status.md) for *where we are*; this folder is *why
we chose this shape*. Numbering is chronological. Next unused number: **0016**.

## Index

| # | Title | Status |
|---|---|---|
| [0001](0001-repo-and-deployment-strategy.md) | Repository & deployment strategy (polyrepo + microservices + Kubernetes) | Accepted |
| [0002](0002-backend-tech-stack.md) | Backend/runtime tech stack | Accepted |
| [0003](0003-dataset-selection-and-synthetic-data.md) | Dataset selection & use of synthesized data | Accepted |
| [0004](0004-modelling-and-label-strategy.md) | Modelling approach & label strategy | Accepted |
| [0005](0005-sentinel-user-exclusion.md) | Data cleaning: excluding non-human sentinel accounts | Accepted |
| [0006](0006-ai-tooling-rules-skill-mcp.md) | AI-assisted development tooling (rules, skill, MCP) | Accepted |
| [0007](0007-evaluation-protocol.md) | Evaluation protocol (label-covered window + chronological split) | Accepted |
| [0008](0008-contracts-freeze.md) | Freeze feature / model / PDP / event / policy contracts | Accepted |
| [0009](0009-online-profile-freeman-serving.md) | Online ProfileState counts + Freeman JSON serving | Accepted |
| [0010](0010-shared-local-data-plane.md) | Shared local data plane lives in `rba-infra` | Accepted |
| [0011](0011-async-outbox-rabbitmq.md) | Phase 4 async plane — outbox → RabbitMQ → profile/audit | Accepted |
| [0012](0012-thin-idp-end-product.md) | Thin IdP end product; PDP remains the risk core | Accepted |
| [0013](0013-idp-staged-start.md) | Start thin IdP in stages; skip Horizon A demo kit | Accepted |
| [0014](0014-thesis-scale-idp-platform.md) | Thesis-scale IdP platform (Authentik/Auth0-shaped, not enterprise) | Accepted |
| [0015](0015-rba-is-the-thesis-core.md) | RBA is the thesis core; the IdP is how it is delivered | Accepted |

## Template

```markdown
# ADR-NNNN: <title>

- Status: Proposed | Accepted | Superseded by ADR-XXXX
- Date: YYYY-MM-DD

## Context
<what forces a decision; constraints; options considered>

## Decision
<what we decided>

## Consequences
<positive + negative results of the decision; follow-ups>
```
