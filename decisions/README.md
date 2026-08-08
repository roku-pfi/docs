# Architecture Decision Records (ADRs)

Each ADR captures one decision: the context that forced it, what we decided, and
the consequences. ADRs are immutable once **Accepted** — if we change our mind, we
add a new ADR that **supersedes** the old one (and mark the old one accordingly),
rather than editing history. This preserves the reasoning trail for the thesis.

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
