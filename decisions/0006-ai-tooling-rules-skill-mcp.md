# ADR-0006: AI-assisted development tooling (rules, skill, MCP)

- Status: Accepted
- Date: 2026-08-08

## Context

Development is AI-assisted (Cursor) across a polyrepo, over many sessions. Without
persistent guidance, each session re-derives the project's invariants (parity, metric
discipline, dataset caveats, commit conventions) from prior chat transcripts, which is
lossy and error-prone. The project also has a thesis-critical requirement to document
continuously, and a growing set of GitHub repos to manage.

Options weighed: do nothing (rely on transcripts); rules only; rules + a documentation
skill; also add MCP integrations now vs later.

## Decision

- **Cursor rules** (`develop/.cursor/rules/`, mirrored to `docs/rules/`): five concise
  always-on/scoped rules encoding project orientation, the feature-parity contract, ML
  data/evaluation discipline, the documentation loop, and git/polyrepo conventions.
- **`/log-progress` skill** (`develop/.cursor/skills/log-progress/`): automates the
  devlog + ADR + findings loop at the end of each step.
- **GitHub MCP** (`develop/.cursor/mcp.json`): remote GitHub MCP server, authenticated
  via a `GITHUB_PAT` env var (no secret committed).
- **Other MCPs (Postgres, Kubernetes, Grafana/Prometheus, MLflow) deferred** to
  Phase 3+ when a running system exists to operate/debug.

## Consequences

- Positive: guardrails and documentation habits persist across sessions and subagents;
  reduced context-rebuilding; tooling decisions are themselves recorded.
- Cost: rules/skill are duplicated (active workspace copy + versioned `docs/` mirror) and
  must be updated in both places. `develop/.cursor/` is not itself a git repo, hence the
  mirror.
- `.cursor/mcp.json` is environment-specific and not mirrored/committed; requires the
  user to create a fine-grained PAT and set `GITHUB_PAT`.
