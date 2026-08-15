# Cursor rules (canonical copies)

Versioned copies of the Cursor agent rules that guide AI-assisted development
on this project. The **active** copies live at the workspace root in
`develop/.cursor/rules/` (that is what Cursor loads); these are the tracked
mirror so the guardrails are part of the thesis record and survive re-clones.

If you change a rule, update **both** places (workspace + here).

| Rule | Applies | Purpose |
|---|---|---|
| [`00-project-overview.mdc`](00-project-overview.mdc) | always | Orientation: repos, phase, stack, current focus |
| [`feature-parity.mdc`](feature-parity.mdc) | always | Train/serve parity contract; excluded features |
| [`ml-data-and-evaluation.mdc`](ml-data-and-evaluation.mdc) | `rba-ml-training/**` | Splits, labels, metrics, dataset framing |
| [`documentation-loop.mdc`](documentation-loop.mdc) | always | Keep devlog / ADRs / findings current |
| [`git-and-repos.mdc`](git-and-repos.mdc) | always | Conventional Commits + polyrepo; no data/secrets |

Related: the `/log-progress` skill
(`develop/.cursor/skills/log-progress/`) automates the documentation loop.
The skill is **not** mirrored here (Cursor-only). ADR-0006 records why rules
+ skill exist.

Each sibling repo also has an `AGENTS.md` for tool-agnostic orientation.
