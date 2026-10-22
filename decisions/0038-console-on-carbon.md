# 38. The console is built on IBM's Carbon design system

- Status: Accepted
- Date: 2026-10-22
- Amends: [ADR-0030](0030-two-themes-one-palette.md) for the console (its own
  component styles give way to Carbon's; the dark theme and the severity ramp
  stand), [ADR-0036](0036-brand-and-per-tenant-login.md) (product finish).
- Related: B-2 (logo blues), U-1 (shell), U-2 (pages).

## Context

After B-2 and U-1 the console worked, but it read as one more generated
dashboard: dark navy with a corner glow, tiny uppercase monospace labels, a
pulsing "live" dot, stat cards with deltas, dot pills, the same rounded box
around everything. Three restyled directions (`design/explorations/`) changed
fonts and colours and kept that feel. The product owner's references (authentik,
Frontegg) feel designed because a system sits under them: authentik's admin is
built on Red Hat's PatternFly; both have a full component vocabulary (avatars,
tags with "+N", overflow menus, expandable rows, pagination, toggles), present
entities as what they are (a face, a logo, a coloured group), and use one shape
language with no decoration.

Options: (1) design our own components from that breakdown; (2) adopt an
established system and brand it. (1) is slower and drifts back to the same
habits; (2) brings rules we did not invent.

## Decision

The console uses **Carbon** (`@carbon/react`, IBM's design system) on its dark
**g100** theme, with RokuSec on top:

- **Colour**: Carbon's blue tokens are replaced by the logo's (`#1466E5` for
  primary buttons, `#4C9BFF` for links, focus borders and interactive accents);
  everything else is g100. Override layer: `admin-ui/src/console.scss`.
- **Type**: IBM Plex Sans and Plex Mono, Carbon's own families, served from the
  IdP's origin (`@fontsource`), not IBM's CDN. Sora stays in the logo.
- **Shell**: Carbon's UI shell. The header names the product the way Carbon
  products do ("RokuSec **Consola**", with the mark); the side navigation groups
  pages by what the operator is doing (Actividad, Directorio, Acceso, Plataforma)
  with the organisation, and its switcher, on top.
- **Presentation**: two-line cells (what it is, then what tells it apart);
  people with an avatar; groups as coloured tags with "+N"; relative time with
  the exact time below; row actions in an overflow menu; destructive actions
  confirmed in a danger modal; a one-time secret shown in a modal with a copy
  button, never a toast.
- **Copy**: Spanish (es-AR), including Carbon's built-in labels that default to
  English (menu, close, pagination).
- Motion is Carbon's own; the only custom motion is the scanning ring on the
  hosted login.

Pages move one at a time; until then a page renders inside `.legacy`, which
confines its old styles.

## Consequences

- The console stops looking generated and gains accessible, keyboard-complete
  components for free.
- It also looks related to IBM products; the brand carries through the logo,
  the blues and the copy, not through custom shapes.
- The console bundle grows (Carbon's CSS is about 1.2 MB, 114 kB gzipped).
  Importing only the components used is a later optimisation.
- The hosted login and the customer apps are not affected by this ADR (U-3,
  U-4, U-5 decide theirs).
