# Mockups

Twelve screens, exported in two formats. Nothing here is written by hand —
everything is generated from `design/rba-screens/` by `export.py`, so the way to
change a mockup is to change the artboard and re-run the script.

```
python3 design/rba-screens/export.py          # HTML + PNG
python3 design/rba-screens/export.py --html   # HTML only, no browser needed
```

## Which format to use

| You are… | Use | Why |
|---|---|---|
| Pasting into the document (Word, Docs, LaTeX) | `png/` | Insert → Image. Nothing to install, nothing to render, no layout surprises. |
| Building slides for the defence | `png/` | Same file, drag straight onto the slide. |
| Reviewing them all at once | `index.html` | Contact sheet — open it in a browser, click any image for full size. |
| Showing one on a screen, or editing the copy | `html/` | One self-contained file per screen. Opens in any browser, no server. |
| Changing the design | `../../design/rba-screens/*.dc.html` | The editable source. `export.py` reads these. |

**PNG is the right answer for the document, and it is worth knowing why it is
not SVG or PDF.** These screens are HTML with web fonts, gradients and shadows;
every route to vector either rasterises them anyway or substitutes the type. The
PNGs are exported at **2×** — an 1140-point-wide console screen lands as a
2280 px file. Placed at ~16 cm on an A4 page that is well over 300 dpi, which is
the only number a printer cares about.

Do not scale a PNG *up* in the word processor. If a screen needs to be bigger
than it exports, raise `SCALE` in `export.py` and re-run.

## The screens

| File | Screen | Evidence for |
|---|---|---|
| `01-login-sign-in` | Hosted login, credentials | RF-01, RF-06 |
| `02-login-step-up` | Step-up challenge after a MEDIUM/HIGH score | RF-05, RF-08, **RF-18** |
| `03-login-blocked` | Blocked attempt, no retry offered | RF-04, RF-18 |
| `04-bank-accounts-customer` | Relying party, `customer` role | RF-21 |
| `05-bank-accounts-readonly` | Relying party, `viewer` role | RF-21 |
| `06-bank-transfer-denied` | 403 on the write path | RF-21 |
| `07-admin-decisions` | Scored logins with per-signal reasons | RF-12, RF-14, HU-10 |
| `08-admin-policy` | Bands, action ladder, monitor-only | RF-03, RF-09, RNF-05, RNF-08 |
| `09-admin-users` | Directory | RF-13 |
| `10-admin-roles` | Roles and `resource:operation` permissions | RF-13, RF-21 |
| `11-admin-groups` | Groups, members, role assignment | RF-13 |
| `12-design-system` | Palette, severity ramp, type scale | Rubric: *Mockups* |

Screens 04–06 are the pair worth putting side by side. Same password, same
device, same country, same risk decision — they differ by one button because the
two users differ by one permission, and 06 is the 403 the server returns when
the viewer skips the button and posts to `/transfer` anyway. That is RF-21 in
three images.

## What is drawn ahead of the code

The mockups are a design document, not a screenshot set, and three things on
them do not exist yet. They are listed here so nobody has to discover it during
the defence:

- **`07-admin-decisions`** — the filter row, the per-signal contribution bars
  (today the reasons render as a plain bulleted list), and the role chip in the
  header.
- **`08-admin-policy`** — the structured band/ladder editor. This tab today is a
  raw JSON textarea over the same bundle. The numbers, the hot reload and the
  `policy:write` guard are all real; only the editing surface is ahead of itself.
- **`09-admin-users`** — the group column.

Everything else on all twelve screens runs today.

For screenshots of the software *as it is*, run the walkthrough in
[`docs/plans/status.md`](../plans/status.md) and capture the browser. Both belong
in the document: the mockups show the design, the screenshots show the state.

## Colour

Severity never uses red or orange. It escalates by moving away from green —
green → cyan → indigo → plum — which separates on lightness as well as hue, so
the ramp survives greyscale printing and red-green colour vision deficiency.
Severity is always an outlined monospace pill; an action you can take is always a
solid filled control. Different shape, so indigo can safely do both jobs.

The light theme is the end user's side, the dark theme the operator's. See
[`ADR-0030`](../decisions/0030-two-themes-one-palette.md).
