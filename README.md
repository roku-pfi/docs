# RBA project documentation

Central, cross-cutting documentation for the risk-based authentication project
(org: `roku-pfi`). Service-specific usage docs live in each service repo's README;
this repo holds the things that span services and feed the thesis report.

## Contents

- **[`devlog.md`](devlog.md)** — chronological development log. What we did each
  session, why, and what we found. Newest entries at the top.
- **[`decisions/`](decisions/)** — Architecture Decision Records (ADRs). One short,
  numbered file per real decision: context, the decision, and its consequences.
  Start at [`decisions/README.md`](decisions/README.md).
- **[`findings/`](findings/)** — analysis / experiment write-ups (EDA, model
  results) with the concrete numbers the thesis will cite.

## How we keep this updated

- After each **step** or work session → add a `devlog.md` entry.
- Whenever we make a **non-trivial choice** (architecture, tooling, data, modelling)
  → add an ADR (or supersede an existing one).
- After each **experiment/analysis** → add or update a `findings/` write-up.

## Related

- Roadmap / overall plan: `plans/development_plan.md` (in the workspace).
- Code repos: `rba-features`, `rba-ml-training`, and the service repos under
  `github.com/roku-pfi`.
