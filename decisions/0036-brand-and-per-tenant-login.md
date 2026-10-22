# 36. RokuSec brand, per-tenant login branding, apps with their own identity

- Status: Accepted
- Date: 2026-10-21
- Amends: [ADR-0030](0030-two-themes-one-palette.md). Its light/dark split, the
  severity ramp (never red) and self-hosted fonts stand; its single shared
  palette across the bank and the IdP does not.
- Related: [ADR-0016](0016-hosted-login-on-idp.md) (hosted login on the IdP),
  [ADR-0023](0023-end-user-login-is-opaque.md), [ADR-0033](0033-multi-tenancy.md).

## Context

The product had no brand: the screens used a generic indigo accent and a shield
glyph, and the demo bank wore the IdP's palette, which made a customer
application look like part of the identity provider. The logo concept (an open
padlock that draws a 6, *roku*) now exists as vector files (`docs/brand/`).

The UIs also read as demos rather than a product: mixed English and Spanish,
raw identifiers and version strings in the body of pages, no app shell, few
empty, loading and error states.

## Decision

### RokuSec surfaces

| Surface | Theme | Who |
|---|---|---|
| Hosted login (RokuSec-branded tenants) | light | Martín, when his organisation uses RokuSec's own login |
| Console | dark | Diego and the platform operator |

The light/dark split exists so the end user's window and the operator's window
are told apart at a glance; it is not a rule for customer applications. The
RokuSec accent moves from indigo (`#4C5FE0`) to the logo's blues
(`docs/brand/README.md`); the severity ramp of ADR-0030 is unchanged. The
logo's wordmark face (Sora) may be used for display text; body text stays
Instrument Sans, data stays IBM Plex Mono.

### Customer applications have their own identity

Roku Bank and Kaze Labs each get their own name, logo, palette and type,
unrelated to RokuSec and to each other. They no longer share `theme.css` with
the IdP; each app owns its stylesheet.

### The hosted login is branded per tenant

The login is always served by the IdP. The password is typed on RokuSec's
origin and never on the application's (ADR-0016). Each tenant chooses how that
page looks:

| Style | Looks like | Used by |
|---|---|---|
| `rokusec` | RokuSec's own login, "Continuá a Kaze Labs" | Kaze Labs |
| `tenant` | the tenant's logo, colours and wording, with a small "Protegido por RokuSec" seal | Roku Bank |

The tenant's branding (display name, logo, two colours, login style) is tenant
configuration in the IdP, edited in the console. The step-up and blocked pages
follow the same branding and keep the non-disclosing copy of ADR-0023.

### Language

Everything a person reads in a UI is Spanish (es-AR), including the console and
both applications; dates and numbers are formatted for es-AR. Logs, metrics,
reason codes and API fields stay in English. Reason codes are translated for
display in the console, with the code shown on hover.

### Product finish

The console gets an app shell (logo, tenant name, navigation, account menu),
list pages with search, filters and pagination, detail pages, toasts, confirmation
dialogs for destructive actions, and empty, loading and error states. Model
versions, IDs and other technical details move into a collapsible
"Detalles técnicos" section instead of the page body.

## Consequences

- ADR-0030's "identical `theme.css` in two repos" duplication disappears; each
  app owns its look.
- The tenant-branded login runs on RokuSec's `*.cloudfront.net` name, not on the
  bank's own domain. A custom domain per tenant is future work and is what would
  complete the illusion.
- The mockups in `docs/mockups/` (12 screens) are regenerated from the rebuilt
  screens for the informe.
- A logo rationale and brand guide go into the informe's branding section.
