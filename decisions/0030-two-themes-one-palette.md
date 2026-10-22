# 30. Two themes, one palette: light for users, dark for operators

- Status: Accepted (shared palette across apps amended by [ADR-0036](0036-brand-and-per-tenant-login.md))
- Date: 2026-08-20
- Replaces the warm serif look that had grown across `login.css`, `home.css` and
  the admin console.
- Relates to: the rubric's *Mockups* concept, and to
  [ADR-0023](0023-non-disclosing-step-up-copy.md) (what a step-up may say).

## Context

Three surfaces had accumulated three copies of the same warm-paper, orange-accent,
Palatino-serif look: the hosted login page, the demo bank, and the admin console.
Each declared its own hex values. Severity used red/amber/green.

Two problems, one cosmetic and one not.

The cosmetic one: the values were duplicated, so a colour meant three slightly
different things depending on which page you were on, and the mockups drawn for
the 50% delivery matched none of them.

The one that matters: **red/amber/green is a bad ramp for this particular
table.** Around 8% of men have red-green colour vision deficiency, and the
person who reads a severity column for an hour chasing one incident is exactly
the person the ramp exists for. Red and green also collapse to nearly the same
grey, so a decision table printed in the thesis loses its ranking entirely.

## Decision

### The audience picks the theme

| Surface | Theme | Who is looking |
|---|---|---|
| Hosted login (`rba-idp`) | light | a customer, a few times a week, on a phone |
| Demo bank (`rba-demo-banking`) | light | the same customer |
| Admin console (`rba-idp/admin-ui`) | dark | an operator, for an hour, chasing one incident |

This is not a user preference toggle and there is no theme switcher. Light is
easier to trust on a page that asks for a password; dark holds up over a long
session and lets the severity chips carry weight against a quiet ground. Picking
per surface means neither audience pays for the other's needs.

### One token file, two palettes

`theme.css` defines the light palette on `:root` and the dark palette on
`:root[data-theme="dark"]`. The console's `index.html` carries
`data-theme="dark"`; nothing else does. Every stylesheet consumes `var(--…)` and
declares no hex value of its own.

`rba-demo-banking` gets its own copy of `theme.css` and the font files, because
it is a separately deployable service that must not depend on the IdP being
reachable to render. The duplication is deliberate and the file is identical.

### Severity escalates away from green, not toward red

| Step | Level | Action | Light | Dark |
|---|---|---|---|---|
| 1 | LOW | `ALLOW` | `#2F8055` green | `#63C48E` |
| 2 | MEDIUM | `REQUIRE_MFA` | `#22809E` cyan | `#69C3DE` |
| 3 | HIGH | `REAUTHENTICATE` | `#5B4BC4` indigo | `#A192F0` |
| 4 | CRITICAL | `BLOCK` | `#8A3A93` plum | `#D08BE0` |

The ramp separates on **lightness as well as hue**, so it still ranks correctly
in greyscale and under either common form of red-green CVD. Nothing in the
system uses red or orange, including error states: a wrong password is not a
severity level and does not borrow the ramp.

### Severity is a pill; an action is a button

The accent is indigo, and step 3 of the ramp is also indigo. That is only safe
because the two never share a shape:

- **Severity** is always an *outlined* monospace pill — a label, not a target.
- **An action you can take** is always a *solid filled* control.

Any future colour added to the ramp inherits this and needs no further thought.

### Two faces, self-hosted

**Instrument Sans** for anything read as language. **IBM Plex Mono** for anything
read character by character: risk scores, permission scopes, event ids, reason
codes, account numbers, model versions. Aligning those in a column is most of
what makes the decision log scannable, and tabular figures are why money reads
correctly in the bank.

The 78 KB of woff2 is **served from our own origin**, not from a CDN. A login
page that links `fonts.googleapis.com` hands Google a request on every sign-in
attempt, including the failed ones — a needless third-party observer on the one
page in the system that exists to handle credentials. Self-hosting also means
the demo works with no network.

## Consequences

**Good.**

- One place to change a colour, and the mockups and the running software now
  agree.
- The decision table ranks correctly in print and under red-green CVD.
- The login page makes no third-party requests at all.
- The console header now lists the permissions the session actually holds, which
  is the on-screen answer to "why can I see this button and my colleague cannot".

**Costs and risks.**

- `theme.css` and the font files exist twice, once per deployable. They must be
  changed together; a test in each repo asserts its own copy is served.
- Anyone rebuilding the admin SPA must re-run `npm run build` — the bundle
  filename is content-hashed and `web/admin/index.html` is committed.
- Dropping red means severity is no longer guessable from convention alone. The
  chips are always labelled with the word, never colour alone, which is the
  correct fix regardless.
- `web/static/admin.css` was restyled with everything else, but the page behind
  it — the no-build `admin.js` fallback — still calls the `/grants` endpoints
  [ADR-0029](0029-rbac-roles-and-permissions.md) removed. It is unreachable
  while the Vite build is committed. It should be deleted, not repainted again.

## Alternatives rejected

- **One theme for everything.** Cheaper, and wrong for one of the two audiences
  whichever one is chosen.
- **A user-facing theme toggle.** Doubles the visual states to check for no
  requirement, and invites a customer to read the console's palette as an
  option rather than a signal.
- **Keep red for BLOCK only.** It is the one place convention is strongest — and
  the one place the ramp most needs to stay legible in greyscale. A ramp with a
  single exception is not a ramp.
- **Google Fonts CDN.** One `<link>` instead of six files, at the cost of a
  third-party request on every login attempt.
