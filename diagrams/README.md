# Diagrams

Six figures: the login flow in two panels, the deployed architecture, and
three relational schemas (IdP directory, IdP session, decision/profile/audit).
Nothing here is drawn by hand — every figure is laid out by a script in
[`design/diagrams/`](../../design/diagrams), so keeping them true to the system
costs a re-run rather than an afternoon in a vector editor.

**The figures are lettered in Spanish**, because they go into the informe. Titles,
captions, phase headings and every explanatory sentence are Spanish; identifiers,
HTTP routes, field names, database and queue names, and protocol values
(`ALLOW`, `MFA_REQUIRED`, `INVALID_CREDENTIALS`) are left exactly as they appear
in the code, because renaming them in a drawing would make the drawing lie. Short
technical role labels on the lifelines (`Browser`, `user agent`, `relying party`)
stay in English for the same reason.

The figures also carry **no references of any kind** — no ADR numbers, no links
into `docs/`, and no requirement identifiers either. Those are all working
notation; a figure that says "RF-09" is asking the reader to hold a numbering
scheme in their head while looking at a drawing. Where a caption used to cite
something it now states the reason itself, and the mapping from figure to
requirement lives in the table below instead, where prose can carry it.

```
python3 design/diagrams/build.py            # SVG + PNG + contact sheet
python3 design/diagrams/build.py --no-png   # SVG only, no browser needed
```

## Which format to use

| You are… | Use | Why |
|---|---|---|
| Writing in LaTeX | `svg/` | Vector. `\includesvg` (needs Inkscape), or convert once to PDF. |
| Writing in Word or Google Docs | `png/` | See the caveat below — the SVG may re-render with the wrong type. |
| Building slides | `png/` | Drag and drop, no conversion step. |
| Reviewing them all | `index.html` | Contact sheet. Click a figure for the vector original. |
| Handing over a UML source file | `puml/` | PlantUML. Renders anywhere; see the caveat below. |
| Changing a figure | `design/diagrams/*.py` | The source. Edit, re-run `build.py`. |

**Unlike the mockups, the source of truth here is SVG, not PNG.** The mockups are
screenshots of HTML and had to be rasterised; these figures are drawn as vector
from the start, and both fonts (Instrument Sans, IBM Plex Mono) are embedded in
each file as base64 `woff2`. Any browser renders them exactly as designed, at any
zoom, from a ~110 KB file.

The caveat is Word and Google Docs. They render SVG through their own engines,
which do not reliably honour an embedded `@font-face`. If type in a placed SVG
does not match the corresponding PNG, that is what happened — use the PNG. The
PNGs are exported at 2×, so `01-login-sequence.png` is 2800 × 5302; placed at any
size that fits an A4 page it is well past 300 dpi.

## The PlantUML copies

Each figure also ships as a `.puml` in [`puml/`](puml/), for a reader who wants
the UML source rather than a picture:

```
plantuml -tsvg docs/diagrams/puml/01-login-sequence.puml
```

Two things to know before using one. They are **hand-written equivalents, not
generated from the Python** — `build.py` only copies them across, so a change to
a figure has to be made in both places or they drift apart. And they lay out
through Graphviz, which means the architecture one in particular comes out
looser and less legible than the drawn version. **The figures in the informe
should be the PNGs.** The `.puml` files are there so that "can we see the UML
source?" has an answer.

The step numbers do match: `01` runs 1 → 32 and `02` picks up at 33, the same as
the drawings, and `autonumber` keeps them in step as messages are added.

## The figures

| File | What it shows | Evidence for |
|---|---|---|
| `01-login-sequence` | Figure 1a — password, application access, the risk decision, and the three ways the IdP enforces it | RF-01, RF-04, RF-05, RF-09, RF-10, RF-12, RF-14, RF-18, RNF-03, RNF-08 |
| `02-session-authorization` | Figure 1b — the one-time code becomes a session; permissions are resolved on every request | RF-13, RF-21, HU-10 |
| `03-architecture` | Figure 2 — what runs in the cluster and what it talks to | the *Arquitectura* / *Tecnologías* sections — no single requirement |
| `04-schema-idp-dir` | Figure 3 — relational schema of `rba_idp` (users, groups, roles, permissions, apps) | the *Modelo de datos* section |
| `05-schema-idp-runtime` | Figure 4 — relational schema of `rba_idp` (sessions, MFA, passkeys, auth codes) | the *Modelo de datos* section |
| `06-schema-ops` | Figure 5 — relational schemas of `rba_decision`, `rba_profile`, `rba_audit` | the *Modelo de datos* section |

The two sequence panels are one flow. Steps are numbered continuously — 1a runs
1 → 32, 1b picks up at 33 — so prose can say "step 19" and mean exactly one
message.

## A note on size

Figure 1a is 32 steps tall, and on an A4 text block it is limited by its height
rather than its width: it places about 130 mm wide, which puts the message
labels near 3.5 pt. Comfortable on screen at any zoom; small on paper.

Splitting it further would not really fix that. A drawing 1400 points wide, laid
out at full A4 text width, tops out around **4 pt** whatever its height — so the
gain from another split is half a point, and the cost is cutting the risk
evaluation in half, which is the part the thesis is about. The split into two
panels was made where the flow has a real seam instead: authentication ends, the
session begins.

Treat these as screen figures — full page, and readers zoom. If the printed copy
matters more than the on-screen one, the fix is not more panels but a shorter
**overview** figure for the running text, with figure 1a moved to an annex. That
figure does not exist yet; it is a small change to `figures.py` if you want it.

## Reading the drawings

Both sequence panels follow the same conventions:

- **Solid arrow, filled head** — a call. **Dashed arrow, open head** — its reply.
- **Indigo** marks the risk decision itself: the call to the PDP and its answer.
- **Green / plum** mark terminal outcomes — a session created, or an attempt
  refused. Severity never uses red or orange anywhere in this project; it
  escalates by moving away from green, so the ramp survives greyscale printing
  and red-green colour vision deficiency.
- **A loop on a lifeline** is work a service does inside itself. When it carries
  a small cylinder chip (`rba_idp`, `redis`, `rba_decision`), that work touches
  the named store. Datastores are deliberately not lifelines: a round trip to
  Postgres costs two rows and says nothing the reader had not assumed. The
  topology is in figure 2 instead.
- **`opt` / `alt` boxes** are UML combined fragments. `opt` is a branch that ends
  the login; `alt` is a choice between the three enforcement outcomes.

The three data-model figures follow a different convention, because they are not
sequences:

- Each box is a table. Types are PostgreSQL. `created_at timestamptz` is on
  every table and is omitted from the drawing.
- **Solid arrow, filled head** — a foreign key, pointing at the referenced
  table. **Dashed arrow** — an identifier stored without a `FOREIGN KEY`
  constraint (`auth_codes.user_id`, `outbox.event_id`, `application_id` on
  sessions). No crow's-foot marks.
- Redis is not drawn: it is a JSON document, not a schema. Its fields are in
  the informe's field table, not in a figure.

## What is drawn ahead of the code

Nothing. Unlike [`docs/mockups/`](../mockups/README.md), every message, payload
field, database name, queue name and routing key in these figures was read out of
the code and the Helm chart while drawing them. Where a payload is drawn without
a field, the Pydantic model does not carry that field.

Figure 2 draws only what is **deployed**. Three repositories are therefore
absent: `rba-contracts` and `rba-features` are libraries, `COPY`-ed into the
service images and `pip install`-ed at build time — they run inside the pods that
are drawn rather than beside them — and `rba-ml-training` is offline work whose
output crosses into the picture as a scoring artefact baked into the
`rba-decision-service` image, not as a running process. Drawing them as boxes in
a deployment diagram would have implied three services that do not exist.

Two more things are worth knowing because they are easy to misread rather than wrong:

- **`rba-audit-service` is deployed but off the console's read path.** ADR-0018
  points the admin console at the PDP's own `decisions` table so a scored login
  appears immediately, without waiting for the async hop. The audit archive still
  fills; nothing reads it by default.
- **The degraded path in figure 1a is not persisted.** When the PDP does not
  answer, the IdP degrades to a step-up, and that decision exists only in the IdP
  log and on the challenge row — the decision store lives behind the service that
  just failed.
