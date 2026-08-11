# AGENTS.md — docs

Central, cross-cutting documentation for a risk-based authentication (RBA) thesis
project (org `github.com/roku-pfi`). This repo is the project's **memory**: roadmap,
status, decisions, and results. Portable orientation for any AI coding tool. (Cursor
users also get the always-on rules in `../.cursor/rules/`, mirrored under `rules/`.)

## Start here (in order)

1. **`plans/status.md`** — single source of truth for **where we are** (phase/step
   checklist + current focus). Update this whenever a step/phase completes.
2. **`devlog.md`** — narrative log, newest entry on top (what/why/findings per session).
3. **`plans/development_plan.md`** — the phased roadmap (§8 = the phase list) and
   architecture rationale.
4. **`decisions/`** — ADRs (one per real decision: context / decision / consequences).
5. **`findings/`** — experiment write-ups with the concrete numbers the thesis cites.

## Polyrepo context

Sibling git repos cloned side-by-side: **`rba-features`** (shared feature library),
**`rba-contracts`** (OpenAPI/AsyncAPI/JSON Schema + Pydantic contracts),
**`rba-decision-service`** (PDP / `/risk/evaluate`),
**`rba-ml-training`** (offline data + modelling pipeline), and this **`docs`** repo.
Code changes live in those repos; anything cross-cutting or thesis-facing lives here.

## The documentation loop (thesis-critical — keep it continuous)

After each step, decision, or experiment, update the relevant artifact — do NOT batch
it for "the end":

- New/finished **step** → tick `plans/status.md` + add a dated `devlog.md` entry.
- Real **decision** → add an ADR in `decisions/` (numbered). ADRs are immutable once
  Accepted; to change course add a new ADR that supersedes the old one.
- **Experiment/analysis** → add a `findings/YYYY-MM-DD-*.md` note (tables preferred).

In Cursor, the `/log-progress` skill automates this loop.

## Conventions

- Only commit when explicitly asked; Conventional Commits (`docs:`, `feat:`, …).
- One repo per commit; a change spanning repos = one commit each.
- Never commit secrets. (This repo holds no datasets or artifacts by design.)
