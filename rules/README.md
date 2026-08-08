# Cursor rules (canonical copies)

Versioned copies of the Cursor agent rules that guide AI-assisted development on this
project. The **active** copies live at the workspace root in `develop/.cursor/rules/`
(that's what Cursor actually loads); these are the tracked mirror so the guardrails are
part of the thesis record and survive re-clones.

If you change a rule, update both places (workspace + here).

| Rule | Applies | Purpose |
|---|---|---|
| `00-project-overview.mdc` | always | orientation: repos, phase, stack |
| `feature-parity.mdc` | always | the train/serve parity contract |
| `ml-data-and-evaluation.mdc` | `rba-ml-training/**` | splits, labels, metrics, dataset framing |
| `documentation-loop.mdc` | always | keep devlog/ADRs/findings current |
| `git-and-repos.mdc` | always | commit + polyrepo conventions |

Related: the `/log-progress` skill (in `develop/.cursor/skills/log-progress/`) automates
the documentation loop.
