# Plans

The roadmap lives here. **Do not treat this folder as a dump of old drafts** —
`development_plan.md` is the canonical plan; `status.md` is the living
checklist.

| File | Role |
|---|---|
| [`status.md`](status.md) | **Start here.** Where we are: phase/step checkboxes + current focus. Update whenever a step completes. |
| [`development_plan.md`](development_plan.md) | Opinionated full plan: architecture, phases (§8), what is in/out of scope. Distinct from the exploratory chat in `../../material_sources/rba_architecture_conversation.md`. |

Narrative of *what happened this session* goes in [`../devlog.md`](../devlog.md).
*Why we chose X* goes in [`../decisions/`](../decisions/). *Numbers* go in
[`../findings/`](../findings/).

## How to use them together

1. Read `status.md` “Current focus” before starting work.
2. If the step is unclear, read the matching section of `development_plan.md`
   (§8 for phase list; IdP stages under Phase 7).
3. When the step finishes: tick `status.md`, prepend `devlog.md`, ADR/finding
   if needed.

## Phase snapshot (see `status.md` for checkboxes)

| Phase | What | State |
|---|---|---|
| 0 | Infra (compose now; k3d/Helm later) | Compose done |
| 1 | Data & model feasibility | Done |
| 2 | Freeze contracts | Done |
| 3 | Request path (PDP) | Done (k8s later) |
| 4 | Async profile/audit | Thin slice done |
| 5 | Observability & load tests | Not started |
| 6 | ML lifecycle + generator | Not started |
| 7 | Thin IdP platform | IdP-6 done; **IdP-7 stretch next** |
| 8 | Report & defense | Not started |
