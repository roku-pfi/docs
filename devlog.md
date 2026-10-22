# Development log

Reverse-chronological. Newest first. Each entry: what we did, why, and findings.

---

## 2026-10-22 (latest) — C-5 done; I-2 written, not applied

**C-5.** Every image was already on `python:3.12-slim` with no per-architecture
downloads, so multi-arch is a `buildx` matter: `scripts/build-images.sh` builds
all six for arm64 (EKS Graviton) and amd64, tags them with each repo's short
SHA (ECR tags are immutable) and pushes with `--push`. All six built; the amd64
IdP was run under emulation and its native wheels and DB-IP lookups worked.

**I-2, code only.** `tofu/runtime/`: NAT gateway and the private default route;
EKS via `terraform-aws-modules/eks` with a Graviton node group, metrics-server
and Pod Identity; the AWS Load Balancer Controller; RDS PostgreSQL 16 and
ElastiCache Valkey 8, both TLS and reachable only from the nodes; every password
generated; the `rba-credentials` Secret the chart now references; and a Job that
creates one database and login role per service, since a private RDS cannot be
reached from a laptop. Validated with `tofu validate`; `plan` waits for an AWS
login, `apply` for a go-ahead.

---

## 2026-10-22 (latest) — C-4: schema and seed run once, and k3d end to end

Every service that owns tables (IdP, PDP, audit, profile) has
`python -m <package>.init`: create the schema and seed, idempotently, and never
over an operator's change (the PDP inserts only tenants with no policy). With
`INIT_ON_STARTUP=true` (the default) services still do it themselves, as on
k3d and compose; the IdP's cloud mode requires it off. The chart gains
`init.mode: job`, which runs the four commands as pre-install/pre-upgrade hook
Jobs and leaves replicas with nothing to race on, and
`credentials.existingSecret`, so a Secret created by OpenTofu replaces the
chart's. k3d keeps `startup`: its Postgres is part of the same release.

Found on the way: the profile service still seeded a home history for
`usr_demo`, and `reset-demo.sh` still reset it. ADR-0033 renamed the users, so
Martín's first local login had no history and scored HIGH. It now seeds
`usr_rb_martin` and `usr_kz_martin`.

**First full k3d run since tenancy** (OrbStack back up): all 13 pods healthy,
DB-IP loaded in the cluster, Kaze's Martín ALLOWed at a near-zero score with
the passkey offer, the bank recording verdict ALLOW / enforced REQUIRE_MFA in
MONITOR, the mock OTP answering 404, and a full break-glass round on the real
stack — CLI code inside the pod, `/activate`, a software passkey, then a real
step-up passing; the code refused on reuse. On a fresh local database nobody
can pass a step-up until they enroll, so the chart's NOTES now say how.

---

## 2026-10-22 — C-3: real addresses get a country and a network

DB-IP Lite (country + ASN, IPv4 and IPv6, CC BY 4.0) is fetched at image build
by `scripts/fetch_geoip.py` and read in memory with `maxminddb`; the TEST-NET
table still answers first, for tests and the local walkthrough. Checked against
the real files: a Telecentro address resolves to AR / AS27747 over IPv4 and
IPv6, an AWS one to AS16509. The two files are ~18 MB, not the ~10 MB ADR-0032
estimated. DB-IP returns 403 to Python's default User-Agent; the script sends
its own. Attribution is on the login page and the console.

The override needed the same treatment as the address in C-2: taken from the
request, `country=AR` would let anyone look like home. Where client signals are
not trusted, an override now counts only inside a signed ticket (HMAC, 10
minutes) minted by `POST /simulation/ticket` for a caller holding
`SIMULATION_SECRET`; the presenter panel (D-1) will mint them. Any login whose
override changed what the address says is sent to the PDP as `simulated`, which
the decision record and the console already show. Locally, overrides behave as
before. IdP 222 tests.

---

## 2026-10-21 — C-2: the scored address is the client's

ADR-0032 asked for the client IP from trusted proxy headers. Reading the code
turned up a larger hole: the hosted page posts the `ip_address` it was given,
and `POST /login` scored whatever the body said. Anyone with a stolen password
could claim the victim's usual address with one request. The IdP also believed
any `X-Forwarded-For`, from any peer.

Now the address comes from the request. Headers count only when the direct peer
is in `TRUSTED_PROXY_CIDRS` (the ALB's subnets). Behind CloudFront the IdP reads
`CloudFront-Viewer-Address`, since the right-most `X-Forwarded-For` hop there is
the CloudFront edge. The body's address is used only where
`TRUST_CLIENT_REPORTED_IP` is on (local, for the walkthrough's stand-in
attacker). Two quieter fixes: in the cloud a private address no longer scores as
the demo home (an unresolved ALB address would have looked usual), and global
IPv6 is no longer classed as unroutable (every IPv6 login scored as home). The
bank stops pinning the home country on its plain login link in the cloud. All
four are required by `DEPLOYMENT=cloud`. IdP 202 tests, bank 30.

---

## 2026-10-21 — C-1: a cloud deployment cannot start on laptop defaults

Every setting was already read from the environment. The risk was the other
way round: every default *works*, so a variable forgotten in the cloud values
would fall back silently to `rba:rba@localhost`, an `http://localhost` WebAuthn
origin, the mock OTP, or seed passwords printed in the repo. `DEPLOYMENT=cloud`
(IdP and bank) now validates at startup and lists every such problem in one
error; the bank's session cookie is `Secure` there. Brokers and databases in the
other services fail loudly on their own, so they were left alone. From this step
on, work lands as one branch and PR per repo.

---

## 2026-10-21 — RokuSec logo, redrawn as vector

Start of the final phase (cloud on AWS/EKS, demos built around Martín and Diego,
product-grade UI); the plan itself is still being written. The logo is the one
piece that depended on nothing else.

The chosen concept was an AI-generated PNG. It is now generated from
`design/brand/build.py` into `docs/brand/`: primary, horizontal, inverse (dark),
mono and favicon variants, SVG + PNG. Geometry was measured off the source PNG
pixel by pixel rather than eyeballed, so the redraw overlays the original. The
wordmark is Sora 700 (OFL) converted to outlines — the closest of eight geometric
sans candidates rendered side by side against the source.

One SVG trap worth recording: `clipPath` ignores strokes. The shackle was first
drawn as a stroked arc and the ring's shadow, clipped to it, leaked into the
arc's interior. The shackle is now a filled outline, which also folds the round
end into the same shape.

Open: the UI accent (`#4C5FE0`, ADR-0030) is indigo, not the logo's blue. To be
settled in the UI rebuild.

**The final-phase plan is written.** `plans/final-phase.md` plus six ADRs:
0031 (AWS: EKS on Graviton in private subnets, RDS, ElastiCache Valkey, RabbitMQ
in-cluster, CloudFront for HTTPS since there is no domain, two Terraform stacks
so an idle month costs about a dollar), 0032 (real client IP and offline
geolocation; the IdP had never geolocated a real address and ignored IPv6),
0033 (multi-tenancy, which E90 had excluded while chapter 4's business model
assumed it), 0034 (monitor mode now keeps the organisation's base policy
instead of letting everyone in; closes RF-08), 0035 (the demo around Martín and
Diego, two tenants, attacker Lambdas in three regions) and 0036 (brand, login
branded per tenant, apps with their own identity, Spanish UI).

One thing found while planning: a login from an AWS region is AS16509, which is
on the hosting/VPN list, so the travel rule skips it by design. The attacker
Lambdas therefore demonstrate "untrusted network", truthfully; impossible travel
stays simulated and labelled as such until a real login from abroad is staged.

**I-1 applied.** AWS access moved off the root user to an IAM Identity Center
admin (`aws configure sso`, profile `rokusec`). OpenTofu instead of Terraform
(Homebrew no longer ships Terraform). `rba-infra/tofu/bootstrap` created the
state bucket; `foundation` created 35 resources: VPC with public/private subnets
in two AZs and a free S3 endpoint, seven ECR repositories, the GitHub OIDC role,
a US$30 budget alert, and three CloudFront distributions whose names are now
fixed — auth `d1fg623tjiucw7`, bank `d3m6kfm1gma0g6`, kaze `d14wc5kuicqxf6`
(`.cloudfront.net`). They answer 502 until the runtime stack exists.

Found while reading the PDP for T-1: `PUT /policy` writes the replica that served
it and nobody else. Fixed as part of tenancy (policies in Postgres).

**T-1, T-3 (PDP side), T-4.** `rba-contracts` 0.9.0: `tenant_id` on evaluate,
decision and event; `TenantPolicy` with `base_policy` + `engine_mode` replaces
`PolicyConfig` and its per-app overrides; the response separates the verdict
(`action`) from `enforced_action`; `Tenant` model with branding and operator
domains. The PDP stores policies in `tenant_policies` with a 5 s per-replica
cache (a test drives two stores on one database and watches the second
converge), seeds missing tenants from `config/policies/`, answers 404 for an
unknown tenant instead of inventing a default, and scopes every decision read
by tenant (another tenant's event is a 404, not a 403). Profile and audit carry
`tenant_id`; audit reads are scoped the same way. Suites: contracts 20, PDP 38,
audit 5, profile 2 (new handler test), publisher 11 — all green.

Worth noting from the run: a first login from a user with no history scores
HIGH, so Roku Bank's engine says REAUTHENTICATE while the bank, in MONITOR,
enforces its usual passkey. That is the monitor arc in one decision.

**T-2, T-5, T-6: the IdP is multi-tenant.** `tenants` and `tenant_domains`
tables; `tenant_id` on applications, users, groups and roles; email unique per
tenant. An app login takes its tenant from the application; the new
`POST /console/login` takes it from the email domain, and an unknown domain, an
unknown user, a wrong password and a user without a console role all return the
same `INVALID_CREDENTIALS` at the same bcrypt cost. The IdP enforces
`enforced_action` and, for a tenant whose engine is `OFF`, never calls the PDP
(it reads the tenant's policy through a 5 s cache; saving a policy drops this
replica's copy). Each tenant has a console application with `tenant-admin` and
`support` roles. The admin API resolves every request to an operator and a
tenant, and every query filters by it; another tenant's object answers 404 like
a missing one. The platform operator (`juan@rokusec.example`) switches tenants
with an `X-RokuSec-Tenant` header, which anyone else gets the same 403 for
whether the tenant exists or not. The seed is the demo cast: Martín and Sofía at
Roku Bank, Diego and Ana operating it; Lucas, Martín and Valen at Kaze Labs.

Two holes found and closed while writing it. A tenant admin can name a
permission `admin:read` on their own bank app; the admin API would then have
accepted bank sessions. The API now requires a session on the operator's own
tenant's console. And `tenants:read` granted to oneself on a tenant console is
ignored: it counts only on the platform tenant's console.

`rba-contracts` gains `ConsoleLoginRequest`. A pre-tenancy database is refused
at startup with a message to reset it (no migrations, ADR-0033). The test
suite sets `BCRYPT_ROUNDS=4`, since each app start hashes the eight seeded
users (27 s → 6 s for the admin file). IdP 149 tests (55 new in
`test_tenancy.py`, including a cross-tenant 404 per admin endpoint; removing a
tenant filter fails nine of them), contracts 20, bank 24. Checked end to end
against the real PDP over HTTP: Kaze's console sees only Kaze's decisions, and
switching Kaze to `OFF` made the next login skip the PDP.

Open: the console screens are the old ones plus a tenant switcher (U-1/U-2);
the hosted login shows the tenant's name but not its branding yet (U-3); the
`05-idp-er` figure predates `tenants`; the first step-up still offers passkey
enrollment (T-7).

**T-7: passkeys only from a trusted context (ADR-0037).** A step-up for a user
with no passkey is now refused (`SECOND_FACTOR_MISSING`, "contactá a soporte"),
with `no_second_factor` on the challenge row and the verdict still recorded by
the PDP. Two doors remain. After a login the engine itself judged usual (verdict
ALLOW, not a fallback, not a monitor-mode ALLOW of a risky verdict, not an
engine that is off), the session may add one passkey within 10 minutes. Or with
an activation code an operator issues from the console: 12 characters, 15
minutes, hash only, single use, a new one voids the old, and it needs the
user's password too (a wrong password and a wrong code answer the same).

Found while doing it, and worse than the hole T-7 was for: `POST /mfa/verify`
accepted the fixed code `000000` in every deployment. Anyone with a password
could pass any step-up with one request, no browser needed. It is now off
unless `MOCK_OTP_ENABLED` and answers 404; only the test suite enables it.

Consequence for the cloud: on fresh databases nobody has a passkey, and the
platform tenant asks for one on every login. The first code therefore comes
from the cluster, `python -m rba_idp.activation juan@rokusec.example --tenant
rokusec`; from there codes are issued in the console. IdP 167 tests (18 new in
`test_enrollment.py`, run with the mock OTP off); disabling the refusal fails
two of them.

---

## 2026-08-21 — Data model for the informe

Closes the *Modelo de datos* rubric row. Two figures, generated the same way
as the other three: `04-datastores` (one Postgres, four databases, Redis as a
JSON document, and the files that are not tables) and `05-idp-er` (crow's-foot
of `rba_idp`). Columns and FKs read off the SQLAlchemy models; Redis fields
read off `ProfileState`. The informe section lives at the end of chapter 3,
after Tecnologías — chapter 4 is still unincluded and was only a stub.

Redis is not an ER. Sessions, MFA challenges and auth codes stay in Postgres;
the cache is the behavioural profile the PDP GETs on `/risk/evaluate`. There
is no `Device`, `LoginAttempt` or `Policy` table, and drawing them would have
been the same class of lie as putting argon2id on the login sequence.

---

## 2026-08-21 — The stack, written down — and the figure was wrong about bcrypt

`docs/tech-stack.md` covers the *Tecnologías* row of the 50% rubric, which asks
for two things: the choice of languages **justified**, and the network
architecture that frames the work. So the document is organised around reasons
rather than a bill of materials — why Python (train/serve parity is an import
statement, not a discipline), why FastAPI over Flask (the OpenAPI documents are
generated, not maintained), why RabbitMQ over Kafka (Postgres is already the
system of record), why WebAuthn over TOTP (a step-up to a phishable factor
concedes the argument), why two Helm charts and two namespaces (the tenant
boundary is deployment topology, not discipline). §13 is the network
architecture: the ingress path, the in-cluster DNS table, the port inventory,
the protocols, and where authorization is actually resolved. The ML stack is out
of scope by request and gets §16 only, to place the boundary.

Everything in it was read out of `pyproject.toml`, the Dockerfiles, the charts
and the cluster definition rather than recalled — which is how the second half of
this entry happened.

**Figure 1a said `verify_password(argon2id)`. The IdP uses bcrypt.**
`rba_idp/passwords.py` imports `bcrypt` and nothing else; `argon2` appears
nowhere in any `pyproject.toml` or any source file. The claim was invented and it
had been sitting in a figure bound for the informe. Corrected in `figures.py`,
in the PlantUML copy, in this log's 50%-delivery entry, and in a comment in
`defense-runbook.html` — four places, all rebuilt and re-verified.

Worth naming the failure rather than just the fix: argon2id is what a *good*
implementation would use, so it read as plausible and survived several passes
over the diagram. The check that caught it was not review, it was going back to
the source to write down a version number. The dummy-hash timing defence in
`passwords.py` is real, and is now described where it actually lives.

---

## 2026-08-21 — The architecture figure draws only what runs, and each figure gets a `.puml`

Two changes, both asked for.

**The "not deployed" band is gone.** Figure 2 carried a dashed zone along the
bottom holding `rba-contracts`, `rba-features` and `rba-ml-training`. The band
was honest — it was labelled *no desplegado* — but it was still three boxes in a
deployment diagram, and a box in a deployment diagram reads as a thing that
runs. Two of those repos are libraries: they are `COPY`-ed into the service
images and `pip install`-ed at build time, so they execute inside the pods drawn
above them, not beside them. The third is offline work whose output enters the
picture as a scoring artefact baked into the PDP image, not as a network hop.
Removing the band cost nothing the figure was actually asserting, and it took
218 px off the artboard (1268 → 1050). The *Por qué el PDP no guarda identidad*
callout that lived beside the band moved into the empty column under *Dos
namespaces, a propósito*, rewrapped to the same 320 px width, so the two
callouts now read as a matched pair. The subtitle was rewritten to match: it
used to count repositories, and now it makes the claim the figure is making —
everything drawn is deployed. Where the three repos went is explained in
`docs/diagrams/README.md`, in prose, where a reader who wonders can find it.

**Each figure now also ships as PlantUML**, in `docs/diagrams/puml/`, because a
committee may want an artefact in a UML source format rather than a picture.
These are hand-written, not generated — `build.py` copies them across, and a
change to a figure has to be made in both places. That is a real drift risk and
it is written into the README rather than hidden. Both sequence panels use
`autonumber` seeded at 1 and 33, so the step numbers match the drawings; that
was checked by rendering and reading the numbers back out of the SVG rather than
by counting in my head — 32 messages in 1a, step 19 the outbox insert the
footnote cites, 33 → 42 in 1b. The architecture one lays out through Graphviz
and comes out visibly looser than the hand-laid version, which is the honest
reason the README says the informe should use the PNGs.

Rebuild is clean: bounds 0, overflow 0 on all three, and the marker sweep for
`RF-`/`RNF-`/`HU-`/`ADR-` still returns 0 across the SVGs, `index.html` and now
the `.puml` sources too.

---

## 2026-08-20 — The figures speak Spanish, and stop citing us

The three figures were drawn in English and cited ADRs in their captions. Both
are wrong for the artefact's actual destination. The informe is in Spanish, and
its reader has never heard of ADR-0027 — a caption that says "ADR-0027 — rules
may only raise the action" is asking them to take an internal document on faith
instead of stating the reason.

So: titles, phase headings, guards and every explanatory sentence are now
Spanish, and each ADR citation was replaced by the claim it was standing in for.
Identifiers, HTTP routes, field names, database and queue names, and protocol
values (`ALLOW`, `MFA_REQUIRED`, `INVALID_CREDENTIALS`) stay exactly as the code
spells them, because renaming them in a drawing makes the drawing lie. Short role
labels on the lifelines (`Browser`, `user agent`, `relying party`) stay too.

Requirement markers (RF-09, RNF-03, RF-21) went the same way in a second pass.
The first cut kept them on the argument that they point outward at the informe's
own numbering rather than inward at this repo — but that is still asking a reader
to hold a numbering scheme in their head while looking at a drawing. The
figure-to-requirement mapping belongs in `docs/diagrams/README.md`, where prose
can carry it; the drawing should just be legible.

Spanish runs 15–20% longer than English, which broke three things the English
text had been hiding:

- **Six bullets overflowed their component boxes** in figure 2 — `rba-profile-service`
  ran 46 px past its own border and into `rba-audit-service`. Caught by a checker
  that walks the emitted SVG, matches each text run to its *tightest* enclosing
  rect, and flags anything that runs past the right edge. The first version of
  that checker reported zero, because it matched the enclosing *namespace* zone
  first; sorting candidate rects by area fixed it.
- **A guard collided with a note.** `[sin rol en esta aplicación]` is twice the
  width of `[no role here]`, so it ran under the note on the first message inside
  the fragment. Fixed in `seqlib.py` rather than by shortening the text: a guard
  chip is 20 px tall and a note sits 23.5 px above its row, so the first row
  inside a combined fragment now clears the guard by 44 px. Costs 76 px of height
  on figure 1a and cannot recur.
- Bounds and overflow both read **0** across all three figures afterwards.

Figure 1a is now 1400 × 2651 (2800 × 5302 at 2×). The contact sheet and
`docs/diagrams/README.md` record the language convention so the next edit does
not quietly drift back into English.

---

## 2026-08-20 — Three figures, generated

The *diagrams* rubric concept had no home in E50. It does now:
[`docs/diagrams/`](diagrams/README.md), built by
[`design/diagrams/build.py`](../design/diagrams/build.py).

**The login flow, in two panels.** Figure 1a runs from an unauthenticated `GET /`
to an enforced decision: the redirect to the hosted login, bcrypt, the
role-on-this-application check, the geo resolution, `POST /risk/evaluate`, the
PDP's whole pipeline (profile → features → Freeman → second opinion and travel
rules → policy → monitor-only → decision + outbox in one transaction), and then
the `alt` over ALLOW / REQUIRE_MFA / BLOCK including the full WebAuthn ceremony.
The two `opt` branches that end a login early — wrong password (with the ADR-0027
report to the PDP) and no role here — are on it, and so is the degraded path when
the PDP does not answer. Figure 1b picks up where the browser still holds only a
one-time code: the back-channel exchange, then `GET /session` resolving
permissions inside the session transaction on every request. Steps are numbered
continuously, 1 → 42, so the report can cite one message.

**Figure 2 is the deployed architecture** — Traefik, the two namespaces, the four
databases in one Postgres, Redis, the outbox → RabbitMQ → profile/audit fan-out,
Prometheus and Grafana, plus the three repos that are source rather than
workloads. Endpoint, queue and routing-key detail sits *inside* the boxes rather
than on the edges: an edge label has to fit the gap between two components, box
text does not, so the drawing stays legible without pushing everything apart.

**Written as code, not drawn.** `design/diagrams/dsl.py` is a small SVG builder;
`seqlib.py` does lifelines, messages, internal actions and UML combined
fragments; `figures.py` and `architecture.py` are the layouts. Hand-positioned
diagrams rot the moment the system changes — these are a re-run. Both fonts are
embedded as base64 `woff2`, so an SVG renders identically anywhere without a
network.

Two decisions worth recording. **Datastores are not lifelines** in the sequence
panels: a round trip to Postgres costs two rows and tells the reader nothing they
had not assumed, so store access is an internal action tagged with a cylinder
chip and the topology lives in figure 2. And **the flow is split in two** because
height is the binding constraint in a printed figure — figure 1a alone is 32
steps, and at A4 text width its labels land near 3.5 pt. The split was made at
the real seam (authentication ends, the session begins) rather than where the
arithmetic wanted it; the README says so, along with the fact that a shorter
overview figure is the actual fix if the printed copy matters more than the PDF.

Found and corrected while drawing: `rba_profile` was about to be captioned
"reserved". The profile-service does write it — processed-event dedupe and
profile history — even though the profile the PDP reads lives in Redis.

---

## 2026-08-20 — The design catches up with the code

Two jobs, both of them about the 50% delivery rather than about behaviour.

**Six more mockup screens.** The canvas at `design/rba-screens` had three login
screens and two console screens. Added Policy (with monitor-only where it
belongs — at the top, not buried in the JSON), Users, Groups, and the bank's
three: accounts as a `customer`, accounts as a `viewer`, and the 403 on
`/transfer`. Those last three are RF-21 in one diff — same password, same
device, same risk decision, and the screens differ by one button because the
users differ by one permission.

**An export the document can actually eat.** `design/rba-screens/export.py`
strips the canvas wrapper off each artboard and writes `docs/mockups/html/`
(standalone, one file per screen) plus `docs/mockups/png/` at 2×, with a contact
sheet. PNG is the answer for Word/Docs — these are HTML with web fonts and
gradients, so every route to vector rasterises them anyway, and 2× clears 300 dpi
at the size a thesis prints them. `docs/mockups/README.md` also names the three
elements drawn ahead of the code, so nobody discovers them during the defence.

**Then ported the palette into the running software**
([ADR-0030](decisions/0030-two-themes-one-palette.md)). The warm serif look is
gone from `login.css`, `home.css` and the console. One `theme.css` per deployable
holds the tokens; light on `:root`, dark on `:root[data-theme="dark"]`, and the
console's index.html is the only thing that sets it.

Findings worth keeping:

- **Red/amber/green was the wrong ramp for a severity column.** ~8% of men have
  red-green CVD, and red and green collapse to the same grey in print — so the
  decision table lost its ranking in the thesis PDF. Green → cyan → indigo →
  plum separates on lightness too. Nothing in the system uses red now, error
  states included: a wrong password is not a severity level.
- **The accent is indigo and so is step 3 of the ramp**, which is only safe
  because severity is always an outlined mono pill and an action is always a
  solid filled control. The shape carries the distinction, not the hue.
- **The fonts are self-hosted** (78 KB of woff2, latin + latin-ext). A login page
  linking `fonts.googleapis.com` hands a third party a request on every sign-in
  attempt, failures included. Asserted by a test in each repo, because a 404 on a
  font is silent — the page still renders, just wrong.
- **`web/static/*` in `package-data` does not reach `web/static/fonts/*`.** Found
  while adding the fonts; the wheel would have shipped without them. Fixed in
  both `pyproject.toml` files.
- **The no-build admin fallback is stale.** `web/static/admin.js` still calls the
  `/admin/api/groups/{id}/grants` endpoints ADR-0029 deleted. It is unreachable
  while the Vite build stays committed, so nothing is broken today — but it
  should be deleted rather than repainted a third time.
- The console header now renders the session's actual permissions as chips. It
  is real data from `GET /session` and it is the on-screen answer to "why can I
  see this button and my colleague cannot".

Also corrected two pieces of stale copy on the login page: "not in a group
granted this application" (ADR-0029 made roles the grant) and "the risk service
did not answer — this IdP fails closed" (ADR-0028 made it degrade to a step-up;
that panel only ever fires when the *IdP* is unreachable).

Tests: 91 in `rba-idp`, 24 in `rba-demo-banking`, both green — 196 across all
eight repos.

**And brought the defense runbook up to date.** It had been written on 08-19 and
described a system three ADRs ago. It now lives in the repo at
[`plans/defense-runbook.html`](plans/defense-runbook.html) rather than only as a
published page, and it gained:

- A fourth entry in "things that will ruin the demo", plus **checkpoint 5**:
  `k3d-up.sh` reuses an existing cluster and Postgres keeps its PVC, so a cluster
  created before ADR-0029 has the RBAC tables and none of the rows. Every login
  returns `ACCESS_DENIED` and it reads on stage as a wrong password. One `psql`
  query catches it.
- **Phase 6, authorization.** Viewer vs customer, the 403 on `/transfer` when you
  skip the button, the support operator whose Save is disabled *and* whose API
  call is refused, and revoking `transfer:create` live with no re-login.
- **Phase 7, break it on purpose.** Scale the PDP to zero and log in anyway
  (7a); flip `monitor_only` and watch `BLOCK` get recorded but not enforced (7b).
  7a carries a warning the runbook needed: a degraded login is *not* written to
  the decision store, and in this cluster the audit read path proxies through
  `decision-service`, so admin is unavailable exactly while you are demonstrating
  the degradation. Say it before someone reads it as a bug.
- Four new failure-mode rows, all of them "you left something on from phase 6 or
  7" — `monitor_only` still true, `transfer:create` still revoked. `reset-demo.sh`
  only resets Redis; it does not touch the directory in Postgres.
- Repainted onto the ADR-0030 palette, and the lede now says plainly which phases
  are verified transcripts (1–5, appendix) and which are derived from the code and
  still need a rehearsal pass (6, 7).

---

## 2026-08-20 (later) — RBAC: the document said roles, the code had a constant

The scope document promises *control de acceso basado en roles*; RF-13 promises
"usuarios, grupos, **roles y permisos**"; RF-21 promises authorization over
resources and operations. What existed was `group_app_grants.permission` pinned
to the literal string `"access"`, plus a `users.is_admin` boolean running a
second, parallel authorization system for the console.

Built the real thing ([ADR-0029](decisions/0029-rbac-roles-and-permissions.md)):
`user → group → role → permission(resource, operation)`, NIST RBAC with user
groups. Four new tables in `rba-idp`; `group_app_grants` removed.

Decisions worth remembering:

- **Roles are scoped to one application**, and `set_role_permissions` returns 422
  if you try to put another application's permission on a role. Without that, one
  tenant's admin could mint rights inside someone else's app — the failure that
  makes an RBAC implementation worthless. Covered by a test.
- **Holding a role *is* the sign-in grant.** No `app:access` permission to forget.
  A role with zero permissions is a valid "may sign in, may do nothing else".
- **Permissions reach relying parties flattened and fresh.** `GET /session`
  returns `application_id` + `permissions` as `"resource:operation"` strings,
  scoped to that session's application, resolved inside the session lookup rather
  than baked into the token. A revoked role stops working on the next request —
  there is a test that revokes mid-session with no re-login.
- **`is_admin` is no longer an authorization input.** Three guards replace it:
  `admin:read` to open the console, `admin:write` to change identity,
  `policy:write` to retune risk. Splitting the last one matters: changing who may
  log in and changing how risky a login must be are different powers. The flag
  survives as a directory label, and setting it through the admin API adds or
  removes the platform-admin role, so it can never lie.
- **The bank enforces on the POST handler**, not only by hiding the button.

The seed is now the worked example of §3.2.1's four actors, and it finally
includes the **operador de soporte** — `support@example.com`, holding
`admin:read` + `decisions:read`. That actor is in the thesis and could not exist
while authorization was a boolean: they could read a decision or run the
platform, nothing in between. On stage the sharper demo is the two bank users:
`demo@` and `viewer@` log in identically, get scored identically, and only one of
them can transfer.

Admin console gained a Roles view (create roles, tick permissions, assign to
groups) and now disables write controls for principals without `admin:write`, so
the UI tells the truth about what you can do. Rebuilt the Vite bundle.

`rba-contracts` **0.8.0 — breaking**: `GroupDetail.grants` → `roles`;
`AppPermission` / `GroupGrantPublic` / `CreateGroupGrantRequest` gone;
`/admin/api/groups/{id}/grants` → `/roles`. One gotcha found while writing the
example fixtures: `scope` started as a pydantic `computed_field`, which serialises
out but is rejected on input under `extra="forbid"` — so a client could not GET a
permission and PUT it back. It is a normal field derived by a validator now, and
a forged `scope` in the payload is overwritten rather than trusted.

**Existing `rba_idp` databases must be re-seeded.** `create_all` adds the tables
but cannot populate them, so a running database has no roles and every login
returns `ACCESS_DENIED`.

New tests: 14 RBAC (resolution, scoping, guards, revocation) + 7 bank enforcement
+ a rewritten groups suite. All suites green: contracts 13, PDP 36, IdP 90,
bank 23, features 17, publisher 11, audit 3, profile 1.

---

## 2026-08-20 — requirements audit, then the two gaps worth closing

Read the E50 scope document against the code. 29 requirements (21 RF + 8 RNF):
11 built, 12 partial, 4 not started, 2 in direct conflict with a shipped ADR.
The RBA core is the strong part — RF-03, RNF-05, RNF-06, RNF-07 are done and
demonstrable. What is thin is the IAM shell: no roles (RF-13/RF-21 —
`GroupAppGrant.permission` is pinned to the constant `"access"`), no LDAP, no
alarms, no IdP metrics, no always-MFA switch, no stable device identifier.

Wrote up the document-side corrections separately in
[`plans/e50-document-corrections.md`](plans/e50-document-corrections.md): four
requirements where the implementation is the better answer and the text should
move to meet it (RF-06 passkeys over TOTP, RF-18 opaque copy over a disclosed
reason, RF-03 four risk levels, RF-04 `REAUTHENTICATE` over "limitar la tasa"),
plus three inconsistencies the document has with itself (LDAP is optional in the
alcance and mandatory in RF-19; three levels vs four; rate-limiting listed twice
and implemented nowhere).

Then built the two gaps that were both cheap and interview-sourced
([ADR-0028](decisions/0028-degrade-to-step-up-and-monitor-mode.md)):

**RF-10 / RNF-03 — the PEP degrades instead of 503-ing.** This was the one that
mattered. `rba-idp` raised HTTP 503 when the PDP was unreachable, which is the
`bloqueo masivo` the requirement explicitly forbids — a risk-engine outage became
an authentication outage for every legitimate user. It now synthesises a decision
locally: `REQUIRE_MFA`, `fallback=true`, one `pdp_unavailable` reason, and the
same passkey ceremony as any other step-up. `PDP_UNAVAILABLE_ACTION` is typed
`Literal["REQUIRE_MFA", "REAUTHENTICATE"]`, so `ALLOW` and `BLOCK` are
unrepresentable — the requirement is enforced by the type, not by a comment. A
wrong password is still `INVALID_CREDENTIALS`; admin read proxies still 503. The
workspace README's "Fail closed" invariant said the opposite and has been
rewritten in all three places it appeared.

**RF-09 / RNF-08 — monitor-only mode.** `PolicyBundle.monitor_only`, per
application, hot-editable through `PUT /policy` (so RNF-05 still holds). The PDP
runs the whole pipeline and returns `ALLOW` with the verdict in
`RiskEvaluateResponse.monitored_action` and a `monitor_only` reason. The stored
row and the published event carry the **engine's** action, not the ALLOW —
RF-09 asks to record the decision, not the non-enforcement — and the reason is
what `_row_to_response` reads back so an idempotent replay allows exactly like
the live call did. `rba_decisions_total` gains an `enforced` label. Monitor mode
deliberately also suppresses `fallback_action`: a scorer outage is not a reason
to start acting on a monitored tenant, and RNF-03's guarantee is kept at the PEP
where the PDP's silence makes monitor mode unknowable anyway.

`rba-contracts` 0.7.0 — additive on both counts (`monitor_only` defaults false,
`monitored_action` defaults None). 7 new monitor tests, 4 new degrade tests; all
suites green (contracts 13, PDP 36, IdP 76, features 17, bank 16, publisher 11,
audit 3, profile 1).

Commits: `rba-features` `00a7d8c`, `rba-contracts` `a347475`,
`rba-ml-training` `cb93c68`, `rba-decision-service` `a06019b`, `rba-idp`
`046af17`, `rba-demo-banking` `4531397`, `rba-event-publisher` `bc3ff9e`,
`rba-infra` `6648876`.

**Also corrected a stale citation.** The 2026-08-08 Step 5 finding and the
docstring in `scoring/logreg.py` both quoted logreg recall@1%FPR **0.500**, while
the deployed `logreg-0.1.0.json` carries **0.3947**. Two different training runs,
the older one being cited in the thesis. Full regenerated table in
[`findings/2026-08-20-step5-rerun.md`](findings/2026-08-20-step5-rerun.md),
including why the freeman and logreg depth tables count `n` differently — they
are two code paths (`train.py:136` uses `cumcount()`, `:167` uses the
`user_login_count` feature, which after ADR-0027 counts successes only). Do not
merge those tables.

---

## 2026-08-19 — k3d rehearsal: the stuffing demo proved nothing

First end-to-end rehearsal of the Demo-1…5 walkthrough on the k3d cluster
rather than compose. Two artifacts confirmed loading inside the container
(`freeman-0.2.0.json`, `logreg-0.1.0.json`, threshold 0.9028) — the Dockerfile
ENV path that compose never exercises — and the ADR-0027 success-only
familiarity rule confirmed through the **async** path: after four wrong
passwords the profile still read `login_count 98`, `countries ['AR']`,
`asns ['7303']`.

Then the rehearsal earned its keep. Both credential-stuffing scenarios ran from
RU / 12389, a context Freeman already scores 0.8920 → CRITICAL → `BLOCK` with
**zero** failed logins. The failed-login bands were never reached, so the demo
never showed `REAUTHENTICATE` and the walkthrough page contradicted its own
"all four PDP actions appear once" line. Moved both scenarios to the user's own
`AR / 7303` with a novel IP; the action now walks
`ALLOW → REAUTHENTICATE → BLOCK` on the failure count alone with `risk_score`
pinned at 0.0000 — a strictly better demonstration of why ADR-0027 exists.
A test pins the context so it cannot be made "vivid" again.

Also found, not yet fixed: `rba-event-publisher` never reconnects to RabbitMQ,
so one broker restart silently wedges all profile updates while still logging
`published N outbox row(s)`. And 12 fired attempts consistently record 11.
Numbers, repro and the reset procedure:
[`findings/2026-08-19-supervised-escalation-and-failed-logins.md`](findings/2026-08-19-supervised-escalation-and-failed-logins.md).

---

## 2026-08-19 — Supervised second opinion + failed-login bands (ADR-0027)

Two gaps found while reviewing the project against the proposal PDF: we served
the *weakest* model at the operating point (Freeman recall@1%FPR 0.105 vs
LogReg 0.500), and `failed_logins_last_24h` was structurally always 0 because
the IdP never told the PDP about a wrong password — so credential stuffing, the
threat the proposal leads with, could not influence a decision, and the demo
only ever showed `ALLOW` and `REQUIRE_MFA`.

- `rba-decision-service` **0.2.0**: `scoring/logreg.py` loads a 1.5 KB JSON
  artifact (mean/scale/coef/intercept + baked 1%-FPR threshold) and scores the
  same feature vector with a dot product — no sklearn, no pickle, no hop. Above
  threshold it escalates to `REQUIRE_MFA` and emits `supervised_second_opinion`.
  `services/escalate.py` is the one action ladder
  (`ALLOW < REQUIRE_MFA < REAUTHENTICATE < BLOCK`); travel/VPN, failed-login
  bands and the supervised opinion all name a floor and can only raise.
  `risk_score` stays Freeman's number — ADR-0004 intact.
- `rba-idp` **0.5.0**: a wrong password is reported to the PDP with
  `login_successful=false`, then answered exactly as before. Best effort — a PDP
  outage must not turn a bad password into a 503. Failed-login bands: ≥3 →
  `REAUTHENTICATE`, ≥10 → `BLOCK`.
- `rba-features` **0.2.0** (the important one): reporting failures naively
  created a **profile-poisoning vector** — four wrong passwords flipped five of
  six novelty signals to "seen before", so failing repeatedly *lowered* risk.
  `update_profile` now lets only a **successful** login establish familiarity
  (same rule the travel anchors already used). Both artifacts refit;
  Freeman is now `freeman-0.2.0.json`.
- `rba-demo-banking` **0.3.0**: `stuffing_burst` (4 failures) and
  `stuffing_lockout` (12) walkthrough scenarios. The kit plays the attacker;
  the presenter still does the real login by hand. All four PDP actions now
  appear in one run.

Cost, stated plainly: the poisoning fix drops LogReg recall@1%FPR 0.500 → 0.395
(Freeman unmoved). Worth it — 4 detections out of 38, all inside the noise band,
against a vector anyone can trigger by typing a wrong password five times.
Deployed detection is still 3.7× Freeman alone.

Finding: [`2026-08-19-supervised-escalation-and-failed-logins.md`](findings/2026-08-19-supervised-escalation-and-failed-logins.md)
— includes why `failed_logins_last_24h` gets a *negative* coefficient when
fitted (most failures are legitimate typos), which is the argument for keeping
the stuffing response a deterministic rule rather than a model input.

**Next:** Phase 8 — report & defense. Still open: adaptive-vs-static comparison
(specific objective 4), signal trust boundary + IdP rate limiting, user research.

---

## 2026-08-17 — Demo-4: WebAuthn passkey for REQUIRE_MFA

The walkthrough still used mock OTP `000000` on hosted login. ADR-0022 wants a
real platform authenticator; ADR-0023 wants opaque “confirm it’s you” copy.

- `rba-contracts` **0.6.0**: additive `POST /mfa/webauthn/options` +
  `POST /mfa/webauthn/verify`. `POST /mfa/verify` mock OTP unchanged.
- `rba-idp` **0.4.0**: first MFA **creates** a platform passkey; later MFA
  **gets**. Completing MFA does not re-score. Hosted login drops the OTP field.
  Tests: software ES256 authenticator + existing OTP tests.
- `rba-infra`: Helm sets `WEBAUTHN_RP_ID=localhost` and
  `WEBAUTHN_ORIGIN=http://localhost:8080` (chart 0.3.1).

No new ADR (implements 0022/0023). rpId is `localhost` (compose `:8001` and
k3d `:8080`). First MFA on a machine will prompt to create a passkey.

**Next:** Phase 8 — report & defense. Optional later: DLQ, event lag, Phase 6,
K8s-3.

---

## 2026-08-16 — Demo-3: presenter walkthrough on the bank

The defense needs four scripted logins without putting a scenario dropdown on
the customer home (ADR-0023 / 0026).

- `rba-demo-banking` **0.2.0** (`791a405`): `/walkthrough` (unlinked) +
  `/walkthrough/start?scenario=` clears the session and redirects to the IdP
  with country/ASN — home AR/7303, new country DE/3320, teleport JP/2516, VPN
  US/13335. Customer `/` stays balances-only. Tests: 10 passed.
- `rba-infra` (`ce78d7c`): Helm `demo` NOTES point at `/walkthrough`.
- `rba-idp` (`9c1f010`): README/AGENTS point next at Demo-4.
- Forum UI skipped (looser `demo-forum-app` policy already exists).

Live script: new country first (cold seed → Freeman, travel quiet) → home
ALLOW → teleport immediately → VPN. Admin Decisions in a second window.

**Next:** Demo-4 — WebAuthn for `REQUIRE_MFA` (opaque copy). Mock OTP stays
for tests.

---

## 2026-08-16 — Demo-2: tenant bank in ns `demo` + home-profile seed

A defense needs a client app that *uses* the platform, and a usual home
profile so Freeman can ALLOW. Scenario chrome waits for Demo-3.

- `rba-contracts` **0.5.0**: application `redirect_uri`; `LoginResponse.redirect_to`;
  `POST /callback/token` (one-time code → session). Not OIDC.
- `rba-idp`: seed `demo-banking-app` callback; private/RFC1918 peers score as
  home TEST-NET `203.0.113.10` (RFC 5737 prefixes are *not* rewritten — Python
  `is_private` includes them). Hosted login redirects on `AUTHENTICATED`.
  End-user panels stay opaque (ADR-0023). Tests: 66 passed.
- `rba-profile-service`: `SET NX` home `ProfileState` for `usr_demo` (AR / ASN
  7303 / 14 days of usual devices) via `update_profile`.
- `rba-demo-banking`: opaque Roku Bank home; browser → IdP; server-side code
  exchange; never calls the PDP. Tests: 5 passed.
- `rba-infra`: Helm chart `demo` in namespace `demo`, Ingress `demo.localhost`;
  platform chart **0.3.0** seeds `SEED_APPLICATION_REDIRECT_URI`.

**Next:** Demo-3 — presenter next-login context (home / new country / teleport / VPN)
via the relying party.

---

## 2026-08-16 — Restage Demo-1…4 around the tenant app (ADR-0026)

Old Demo-2 (scenario picker) sat *before* any client app because the walkthrough
lived on hosted login. The product is a bank in ns `demo` that authenticates
through the IdP.

- [ADR-0026](decisions/0026-restage-demo-around-tenant-app.md): Demo-1 stays
  **platform RBA** (done). Demo-2 is **tenant app + seed**. Demo-3 is
  **walkthrough controls**. Demo-4 unchanged (WebAuthn).
- Current focus: Demo-2.

**Next:** `rba-demo-banking` in `demo` + seed `demo@example.com` home profile.

---

## 2026-08-16 — Demo app in another namespace (ADR-0025)

The tenant app should not share the platform namespace. `rba-demo-banking`
deploys in **`demo`**; IdP/PDP stay in **`rba`**. The bank uses RBA by
**redirecting the browser to the IdP**, not by calling the PDP in-cluster.

- [ADR-0025](decisions/0025-demo-app-separate-namespace.md) amends 0024/0020.
- Demo-3 checkbox / catalog updated. Demo-2 still first.

**Next:** Demo-2 — seed + scenario control.

---

## 2026-08-16 — Demo client is a separate app (ADR-0024)

The walkthrough should be a bank that **uses** the RBA platform, not a mocked
`/app` route on the IdP. Colocation (ADR-0022 Demo-3) hid the product split.

- [ADR-0024](decisions/0024-separate-demo-app.md): Demo-3 is `rba-demo-banking`
  (own repo/origin). Redirect to hosted login; IdP asks the PDP; thin
  `redirect_uri` return — not OIDC, not the app calling Freeman. Opacity
  (ADR-0023) still applies on that origin.
- `plans/status.md` / `development_plan.md` §4.1 catalog + Horizon D Demo-3.

**Next:** Demo-2 (seed + scenario control) still first; do not skip it to
scaffold the banking UI.

---

## 2026-08-16 — Demo-1: country on the login path + travel rule

Hosted login was scoring without `country`, so the walkthrough could not show a
physics check. Demo-1 puts country/ASN on the PEP path and a country-centroid
travel **escalate** on the PDP — not a Freeman input (ADR-0022).

- `rba-features` **0.1.2**: `compute_travel(event, profile)` with a static ISO
  centroid table (~1000 km/h). Missing country never fires. VPN/hosting ASNs
  skip teleport, emit `vpn_or_hosting`, and do not move
  `last_login_country` / `last_success_login_ts` (successful residential logins
  only). `FEATURE_NAMES` unchanged. Tests: 17 passed (parity + travel).
- `rba-decision-service`: after `apply_policy`, ALLOW → `REQUIRE_MFA` when
  travel or VPN fires; reason prepended. Score/level stay Freeman's. Tests:
  15 passed, 1 skipped.
- `rba-idp`: boot payload + `POST /login` fill country/ASN from RFC 5737
  TEST-NET prefixes (`203.0.113.0/24` → AR, `192.0.2.0/24` → JP,
  `198.51.100.0/24` → DE) or `?country=&asn=` override. Tests: 57 passed.

No new ADR (implements 0022). No MaxMind / city GPS.

**Next:** Demo-2 — seed `demo@example.com` home profile + next-login scenario
control (home / new country / teleport / VPN).

---

## 2026-08-16 — End-user login is opaque (ADR-0023)

The banking home must not show the PDP decision. Explainability is for the
operator (admin Decisions), not the account holder.

- [ADR-0023](decisions/0023-end-user-login-is-opaque.md) amends Demo-3 in
  ADR-0022 and the end-user reading of ADR-0016.
- `/app` is a normal app. Hosted `ALLOW` redirects with no reason panel;
  MFA/BLOCK use generic copy. Walkthrough keeps admin in a second window.

**Next:** Demo-1 (unchanged) — country on the login path + travel rule.

---

## 2026-08-16 — Horizon D: product demo over GitOps (ADR-0022)

Architecture (k3d/Helm/HPA/Grafana) is enough for the ops story. Current focus
moves from K8s-3 to a thin **product demo** so a defense shows what RBA *is*.

- [ADR-0022](decisions/0022-product-demo-over-gitops.md): Demo-1…4; Freeman +
  policy still decide; the app never calls the scorer. Country-centroid
  impossible travel as a PDP escalate (VPN/hosting skip). WebAuthn for step-up.
  Banking UI colocated on `rba-idp` `/app` — no new service.
- `plans/status.md` / `development_plan.md` §1.1 Horizon D + §8 stages.
- Feature-parity rule: GPS/city still out; centroid travel rule allowed.

**Next:** Demo-1 — country on the login path + `impossible_travel` in
`rba-features` + PDP escalate + parity tests.

---

## 2026-08-16 — K8s-2 Prometheus/Grafana + HPA load (ADR-0021)

Phase 5 thin slice: scrape the PDP, chart it, and prove the HPA moves.

- `rba-decision-service`: `GET /metrics` (`prometheus-client`) —
  `rba_decisions_total`, `rba_risk_score`, `http_request_duration_seconds`.
  Tests: 12 passed, 1 skipped.
- `rba-infra` Helm chart **0.2.0**: Prometheus, Grafana (`/grafana`),
  namespace-scoped kube-state-metrics. Prometheus scrapes **each PDP pod**
  (Endpoints SD). HPA scale-up stabilization 0s; Deployment omits
  `spec.replicas` so Helm does not fight the scale subresource.
- `scripts/load-hpa.sh` + `load/evaluate_load.py`: in-cluster Job, 24
  workers, 180s. **HPA 2 → 5** in ~56s (CPU 9% → 850% of request). Job
  **76177 ok / 0 err**, ~423 req/s. Finding:
  [`findings/2026-08-16-k8s2-hpa-load.md`](findings/2026-08-16-k8s2-hpa-load.md).

**Next:** K8s-3 — GitOps / reusable CI templates (Tilt optional). Event lag
and DLQ stay later. *(Superseded the same day: ADR-0022, Demo-1…4 first.)*

---

## 2026-08-16 — K8s-1 local k3d + Helm (ADR-0020)

The IdP shell is done; this session is the first graded Kubernetes slice.
Compose stays the no-cluster inner loop. A k3d cluster + one Helm chart in
`rba-infra` runs the data plane and every application Deployment.

- `rba-infra`: `helm/rba` (Redis/Postgres/RabbitMQ + apps), `k3d/cluster.yaml`
  (host **8080→80**), `scripts/k3d-up.sh` / `k3d-down.sh`.
- Dockerfiles in `rba-decision-service`, `rba-idp`, `rba-event-publisher`,
  `rba-profile-service`, `rba-audit-service` (sibling libs copied at build;
  audit image serves consumer + `rba-audit-api`).
- IdP is the only Ingress; PDP is ClusterIP at 2 replicas with HPA (min 2 /
  max 5, CPU 70%) and `/healthz` probes. `PROFILE_WRITE_MODE=none`.
- Bring-up: Docker/OrbStack, `k3d`, `helm`, `kubectl`. Cluster **is up**: all
  pods Running, IdP Ingress on `:8080`, HPA seeing CPU. Hosted login + admin
  Decisions exercised (`demo@example.com` → PDP row visible).

**Next:** K8s-2 — Prometheus/Grafana + a load test that moves the HPA (Phase 5).

---

## 2026-08-15 — IdP-7 groups / app-scoped access (ADR-0019)

Login is no longer “any user, any app”. After password verify the IdP requires
a group grant (`permission=access`) for the requested application, then calls
the PDP. Missing grant → `ACCESS_DENIED` (HTTP 200, no score) — not a risk
`BLOCK`. Admin console still uses `is_admin`.

- `rba-contracts` **0.4.0**: `LoginOutcome.ACCESS_DENIED`; group CRUD models;
  `openapi/idp-admin.yaml` `/admin/api/groups*`.
- `rba-idp`: `groups` / `group_memberships` / `group_app_grants`; seeded
  `grp_banking` and `grp_operators`; Groups tab in admin; hosted “no access”
  panel.
- Tests: contracts + IdP login/admin/groups.
- Commits: `rba-contracts` `dedbc4d`, `rba-idp` `e8b15ab`.

**Next:** k8s/Helm/observability (Phase 0 remainder / Phase 5). No OIDC/SAML/SCIM.

---

## 2026-08-15 — Decision browser reads the PDP table (ADR-0018)

Admin Decisions was empty after a real login: it waited on outbox → RabbitMQ →
audit consumer. The PDP already writes `decisions` (with reasons) on evaluate.

- `GET /decisions` on `rba-decision-service` (same contract as audit).
- IdP BFF default `AUDIT_BASE_URL=http://localhost:8000`.
- Rebuild admin SPA empty-state copy.

**Next:** IdP-7 stretch (groups), or k8s.

---

## 2026-08-15 — IdP-6 admin console (`rba-idp` + PDP policy + audit HTTP)

Operators can browse **users, applications, decisions (with reasons), and
policy** without a new `rba-admin-api` / `rba-frontend` repo
([ADR-0017](decisions/0017-admin-console-colocated-on-idp.md)).

- `rba-contracts` **0.3.0**: admin models + `openapi/idp-admin.yaml` /
  `openapi/audit.yaml`; additive `GET`/`PUT /policy` on the PDP spec.
- `rba-decision-service`: `GET`/`PUT /policy` hot-reloads in-process and
  persists YAML. Evaluate path unchanged.
- `rba-audit-service`: FastAPI read API on :8002 (`GET /decisions`). Consumer
  loop unchanged (`rba-audit-service` vs `rba-audit-api`).
- `rba-idp` BFF at `/admin/api/*` (admin Bearer session, `is_admin` flag).
  React (Vite) SPA at `GET /admin`. Seeded operator
  `admin@example.com` / `admin-password`, app `idp-admin-console`.
- Tests: contracts 12 passed; decision-service 9 passed (1 skipped); audit 3
  passed; `rba-idp` 45 passed.

**Next:** IdP-7 stretch (groups) — skippable. k8s/observability are the other path.

---

## 2026-08-15 — IdP-5 hosted login UI (`rba-idp`)

Apps send users to the IdP origin instead of posting JSON themselves.
`GET /` and `GET /login` serve a Jinja page that calls the existing login API
and renders PDP reasons on MFA / reauth / block / success.

- Same origin as `POST /login` — no CORS, no OIDC, no `rba-frontend` yet
  ([ADR-0016](decisions/0016-hosted-login-on-idp.md)).
- `application_id` query param (default seeded `demo-banking-app`); unknown
  clients get an HTML panel instead of a 400.
- Boot payload injects client IP (`X-Forwarded-For` or peer) so the browser
  does not have to guess `LoginRequest.ip_address`.
- Session token stays Bearer in `sessionStorage`; JSON API unchanged.
- `pytest -q`: 34 passed (including `tests/test_hosted_login.py`).
- `rba-idp` `7ff7d21`.

**Next:** IdP-6 — admin console (users, apps, decisions, policy).

---

## 2026-08-15 — Polyrepo README pass

Read every checkout in `develop/` and expanded documentation so each directory
has an accurate README (layout, APIs, env, status). No behaviour change.

- Workspace map: `develop/README.md` (polyrepo is not a single git repo).
- Service READMEs: features, contracts, decision-service, IdP, event-publisher,
  profile-service, audit-service, infra, ml-training (`data/README.md` added;
  the old `data/README.md` link was broken).
- Docs indexes: `docs/README.md`, `plans/`, `findings/`, `rules/`;
  `material_sources/README.md` points at the proposal PDF vs the living plan.

**Next:** IdP-5 — hosted login UI.

---

## 2026-08-15 — IdP-4 session + mock MFA (`rba-idp`)

Login now finishes the PEP loop: `ALLOW` mints an opaque bearer session;
`REQUIRE_MFA` / `REAUTHENTICATE` mint a `challenge_id`; `BLOCK` stays rejected
with neither. Completing the mock OTP (`000000`) issues the same session
without calling the PDP again.

- `POST /mfa/verify`, `GET /session` (`Authorization: Bearer`), `POST /logout`
  (204, idempotent).
- Session tokens stored hashed; challenges expire (5m) and cannot be reused.
- Wrong OTP → `INVALID_CREDENTIALS` (challenge stays open). Unknown/expired
  challenge → HTTP 400. Missing/expired session → HTTP 401.
- No hosted HTML — that is IdP-5.

**Next:** IdP-5 — hosted login UI.

---

## 2026-08-15 — IdP-3 PDP enforce (`rba-idp` → `/risk/evaluate`)

After a successful password verify, `rba-idp` calls `POST /risk/evaluate` and
maps the PDP action onto the login outcome via `outcome_from_action`. Reasons
travel back to the client. This is where RBA re-enters the login path
(ADR-0015).

- `ALLOW` → `AUTHENTICATED`; `REQUIRE_MFA` → `MFA_REQUIRED`;
  `REAUTHENTICATE` → `REAUTH_REQUIRED`; `BLOCK` → `BLOCKED`.
- Invalid credentials / unknown user still skip the PDP (password never leaves
  the IdP). Unknown app remains HTTP 400.
- PDP down or invalid body → HTTP 503 (fail closed; not a fabricated `BLOCK`).
- No session cookie and no `challenge_id` yet — those are IdP-4.

**Next:** IdP-4 — session on `ALLOW`; mock MFA/reauth challenge.

---

## 2026-08-15 — IdP-2 identity store (`rba-idp`)

New `rba-idp` service: local users + one registered application, `POST /login`
password verify (bcrypt). No PDP call, no session, no MFA — those are IdP-3/4.

- DB `rba_idp` added in `rba-infra` init (database-per-service).
- Seed: application `demo-banking-app`, user `demo@example.com` / `demo-password`
  (`usr_demo`) — matches `rba-contracts` login examples.
- Valid credentials → `AUTHENTICATED` + `user_id`. Wrong password / unknown user
  → `INVALID_CREDENTIALS` (HTTP 200). Unknown app → HTTP 400.
- `/mfa/verify`, `/session`, `/logout` are not implemented yet (404).

**Next:** IdP-3 — call `/risk/evaluate`, map action → outcome + reasons.

---

## 2026-08-15 — RBA is the thesis core (ADR-0015)

The Auth0/Authentik-shaped IdP is the **shell**. The thesis is still
**risk-based authentication**: explainable score + action + reasons on a real
login. An IdP that never calls the PDP (skipping IdP-3) would miss the claim.
Admin must show decisions/reasons, not only users.

**Next:** IdP-2 (identity store), then IdP-3 (wire RBA) without skipping.

---

## 2026-08-15 — IdP = thesis-scale Auth0/Authentik (ADR-0014)

Clarified the product metaphor: not a JSON wrapper around the PDP, and not a
carrier-grade IdP. **Like** Auth0/Authentik (users, registered apps, hosted
login, session, MFA, admin). **Unlike** them (no full OIDC/SAML, SCIM, LDAP,
social login). Differentiator remains explainable RBA via the existing PDP.

IdP-2 now includes a seeded **application** (client). Stages otherwise unchanged.

**Next:** IdP-2 — `rba-idp` users + application, password verify only.

---

## 2026-08-15 — IdP staged; IdP-1 contracts only (ADR-0013)

Discarded the Horizon A demo kit. Phase 7 is split into IdP-1…7; this session is
**only IdP-1**.

- ADR-0013: skip demo-kit polish; start the thin IdP one stage at a time.
- `rba-contracts` **0.2.0**: `LoginRequest` / `LoginResponse` / MFA / session.
  PDP `/risk/evaluate` unchanged. Optional risk/session fields stay unset until
  IdP-3/4.
- `status.md` restated as **end product first**: each session advances one IdP
  stage; leftover Phase 4/5/6/0 work is not the current path.

**Next:** IdP-2 — `rba-idp` + `rba_idp` DB, password verify only.

---

## 2026-08-12 — Product target: thin IdP (Oct) on PDP core (demo now)

Locked dual horizon in the **canonical plan** + ADR-0012:

- **Horizon A (~10-day demo):** existing PDP + async plane; no IdP yet.
- **Horizon B (late Oct):** `rba-idp` as PEP (login/session/MFA) calling
  `/risk/evaluate`; admin for users/decisions/policy; **groups/permissions as
  stretch**.
- **Migration rule:** near-term work stays additive — do not put identity in
  `decision-service`; do not treat the demo PEP as the final product shell.

Updated: `plans/development_plan.md` (§0, §1.1, topology, services, Phase 7,
cut order), `plans/status.md`, `decisions/0012-thin-idp-end-product.md`.

**Next:** polish Horizon A for the near demo; IdP only after that.

---

## 2026-08-11 — Phase 4 thin slice: outbox → RabbitMQ → profile/audit

Async plane stood up (ADR-0011):

| Piece | Repo |
|---|---|
| RabbitMQ in shared compose | `rba-infra` |
| Outbox drainer | `rba-event-publisher` |
| Redis profile materialiser | `rba-profile-service` |
| Audit store | `rba-audit-service` |
| `login` on `DecisionMadeEvent` | `rba-contracts` 0.1.1 |

E2E: evaluate with `PROFILE_WRITE_MODE=none` → publisher → Redis profile
`login_count=1` + audit row. Old outbox rows without `login` are skipped by
profile (audited still).

**Next:** remotes/commits for the new repos; then Phase 5 observability or DLQ hardening.

---

## 2026-08-11 — Phase 0 light: `rba-infra` shared Redis/Postgres compose

Moved the local data plane out of `rba-decision-service` into a new `rba-infra`
repo (ADR-0010): one Redis, one Postgres server, init DBs
`rba_decision` / `rba_profile` / `rba_audit`. Service repos consume URLs only.
Full k3d/Tilt/Helm still deferred.

**Next:** run decision-service against this stack end-to-end; then Phase 4 async.

---

## 2026-08-11 — Phase 3 thin slice: `rba-decision-service`

Stood up the request-path PDP against the Phase 2 contracts (ADR-0009):

| Piece | Where |
|---|---|
| `POST /risk/evaluate` | `rba-decision-service` FastAPI |
| ProfileState + Freeman counts + Redis JSON | `rba-features` 0.1.1 |
| Freeman online (`score_event`, JSON export, β=5) | `rba-ml-training` + `artifacts/freeman-0.1.0.json` |
| Policy / reasons / decision+outbox | evaluate service + SQLAlchemy |

Cold-start users correctly get logrisk≈0 (Dirichlet prior = population). Sync
profile writes (`PROFILE_WRITE_MODE=sync`) keep demos working until Phase 4
profile-service. Tests: features 7 passed; decision-service 6 passed / 1 skipped.

**Next:** exercise compose Redis/Postgres day-to-day; then Phase 4 outbox drain.

---

## 2026-08-08 — Phase 2: freeze contracts (`rba-contracts` v0.1.0)

Created the new polyrepo library `rba-contracts` and locked the five Phase-2
artifacts (ADR-0008):

| Contract | Where |
|---|---|
| Feature schema `FeatureVectorV1` (1.0.0) | `schemas/feature-vector.schema.json` + Pydantic |
| Model I/O (`risk_score∈[0,1]`, contributions, artifact sidecar) | `schemas/model-*.schema.json` |
| `POST /risk/evaluate` (PDP) | `openapi/risk-evaluate.yaml` |
| Outbox/bus event `rba.decision.made.v1` | `asyncapi/decision-events.yaml` |
| score→level / level→action policy config | `schemas/policy-config.schema.json` + example YAML |

Key freezes vs the exploratory sketches: probability scale (not 0–100); actions
`ALLOW|REQUIRE_MFA|REAUTHENTICATE|BLOCK`; structured reasons; `event_id` as the
idempotency key end-to-end; Freeman maps `logrisk→[0,1]` via declared
`proba_mapping` (default `logistic_logrisk`); `is_attack_ip` / RTT / absolute geo
kept off the v1 API. `FEATURE_NAMES` here must match `rba-features` (parity test
included). `pytest`: 8 passed.

**Next:** Phase 3 — stand up `rba-decision-service` against these contracts
(Redis profile → features → Freeman inline → policy → decision+outbox).

---

## 2026-08-08 — Freeman calibration (smoothing tune; supervised weighting rejected)

Tried to lift the primary Freeman scorer's weak strict-FPR recall (Step 5: 3/38 @ 1%
FPR) via `ml/calibrate.py` — three levers on the same split, with per-feature LLR
contributions now exposed (`FreemanScorer.contributions_frame`, also future online
explanations). Findings 2026-08-08-freeman-calibration.md.

**Adopted:** lower the Dirichlet smoothing `beta` 10 → 5 (new `FreemanScorer` default).
Label-free, principled (trust a user's short history sooner), improves both ROC-AUC
(0.762 → 0.779) and recall@1%FPR (3/38 → 4/38). **Rejected:** dropping the near-unique
IP (helps operating point but hurts AUC 0.762 → 0.711) and a supervised logistic
per-feature weighting (overfits ~103 train positives → ROC 0.67, recall 0.026). Key
structural note: every variant detects 0 positives in the 0- and 1-2-login buckets —
Freeman's recall is capped by history depth, not feature weighting (logins-to-protection).
Deltas are ±1 detection (38 positives), so the beta change is adopted on principle, not
significance. Refines ADR-0004.

**Next:** Phase 2 — freeze the feature/model/scoring + `/risk/evaluate` contracts.

---

## 2026-08-08 — Step 6: `is_attack_ip` leakage A/B (negative result)

Ran the mandatory leakage experiment (`ml/leakage.py`): each supervised baseline trained
twice on the same chronological split — Variant B (behavioural `rba_features` vectors
only) vs Variant A (B + `is_attack_ip`) — with paired bootstrap CIs (findings
2026-08-08-step6-leakage.md).

**Result: no material leakage.** In this subset the flag is weak/noisy, not an oracle:
`is_attack_ip`=1 on only **47% of takeovers but 8.45% of legit** logins, and the flag
alone scores ROC-AUC 0.70 with **recall@1%FPR = 0**. Our best honest model (logreg)
gains nothing from it — ΔROC-AUC +0.003 (CI [−0.02, 0.03]), Δrecall@1%FPR −0.053. So the
Step 5 headline (logreg 0.88 ROC-AUC, 50% recall @ 1% FPR) is genuinely behavioural, not
inflated by the attack-derived label. Decision (executing ADR-0004): keep `is_attack_ip`
out of the trained model — now evidence-backed, not just precaution.

Refactored `ml/train.py` to share a `build_models(ytr)` factory so Variant B reproduces
Step 5 hyper-parameters exactly.

**Next:** Phase 1 wrap-up — optionally Freeman calibration to lift low-FPR recall, then
lock the feature/model/scoring contracts (Phase 2) before the request path.

---

## 2026-08-08 — Dataset sufficiency note (for the report)

Captured the reasoning on whether the Wiefling dataset is "good enough"
(findings/2026-08-08-dataset-sufficiency.md): it is valid (standard benchmark, faithful
structure) but positive labels are structurally scarce — **141 takeovers is the global
ceiling**, not a sampling artifact, and the scarcity is realistic. Sufficient for
proving the mechanism + driving the system engineering; thin only for tuned detection
metrics, which we handle via the label-free Freeman primary, `is_attack_ip` as a
populous auxiliary signal, a later synthetic scenario generator (complement), and
bootstrap CIs on metrics.

---

## 2026-08-08 — Step 5: Freeman scorer + baselines (feasibility numbers)

Implemented the Freeman likelihood-ratio scorer (`ml/models/freeman.py`), the baseline
trainer/evaluator (`ml/train.py`), RBA metrics (`ml/metrics.py`), and the shared
feature-matrix builder (`ml/featurize.py`).

Key data finding: **takeover labels only cover Feb–Nov 2020** (0 positives afterwards),
so a calendar split gave a test set with 0 positives. Fixed by restricting to the
label-covered window and splitting chronologically within it (ADR-0007). Test set:
45,928 logins / 38 positives.

Results (findings/2026-08-08-step5-baselines.md): the RBA approach works — ROC-AUC
0.76–0.88. Simple **LogReg on `*_seen_before` features hits 50% recall @ 1% FPR**;
Freeman ranks at AUC 0.76 label-free but over-flags new-IP legit logins at the strict
operating point (calibration is the next improvement). Trees underperform LogReg.
Detection concentrates in users with ≥1 prior login (history-depth story confirmed).

**Next:** Step 6 — `is_attack_ip` leakage A/B comparison + finalise metrics/latency.

---

## 2026-08-08 — AI-assisted development tooling (rules, skill, MCP)

Set up persistent tooling so guardrails and documentation habits survive across sessions
(ADR-0006). Added five Cursor rules in `develop/.cursor/rules/` (project overview,
feature-parity contract, ML data/evaluation discipline, documentation loop, git/polyrepo
conventions), mirrored to `docs/rules/`. Added the `/log-progress` skill to automate the
devlog+ADR+findings loop. Wired the GitHub MCP in `develop/.cursor/mcp.json` (auth via a
`GITHUB_PAT` env var). Other MCPs (Postgres/k8s/Grafana/MLflow) deferred to Phase 3+.

**Next:** Step 5 — Freeman likelihood-ratio scorer + reference baselines on a
chronological 70/15/15 split.

---

## 2026-08-08 — Step 4: feature library implemented & validated

Implemented the Phase 1 feature set in `rba-features`: `profile.ProfileState`
(per-user accumulator), `features.compute_features`/`update_profile` (10 features:
`user_login_count`, `{ip,asn,country,device_type,os,browser,hour}_seen_before`,
`seconds_since_last_login`, `failed_logins_last_24h`), and the `replay_user` driver.
Missing values ("-"/NaN) never count as "seen". 5 tests pass, incl. offline↔online
parity. RTT and absolute geo distance excluded (EDA: RTT ~94% missing, geo synthetic).

Validated on the real subset (findings/2026-08-08-step4-feature-validation.md): the
`*_seen_before` features strongly separate takeovers from normal logins
(ip_seen_before 0.08 vs 0.45; country 0.18 vs 0.75) — confirms the RBA hypothesis
and the Freeman-primary direction.

**Next:** Step 5 — Freeman likelihood-ratio scorer + reference baselines on a
chronological split.

---

## 2026-08-08 — Phase 1 kickoff: environment, dataset, subset, EDA

**Environment decision.** Consolidated all development onto the M2 Pro MacBook
(dropped the plan to train on a separate Windows/RTX 3070 PC): GBDT on tabular data
is CPU-bound and barely benefits from GPU, and one machine removes the two-machine
git-sync overhead. See ADR-0002 note.

**Tooling set up (local).** Installed `libomp` + Python 3.12.13 via Homebrew; created
`rba-ml-training/.venv` and installed deps. Verified imports: pandas 3.0.5, numpy
2.4.6, scikit-learn 1.9.0, lightgbm 4.7.0, xgboost 3.4.0, shap 0.52.0, rba_features
0.1.0 (editable).

**Dataset.** Downloaded the Wiefling RBA dataset (Zenodo 6782156, `rba-dataset.csv`,
8.4 GB / 16 cols). Header matches `rba_features.schema` exactly. Documented its
synthesized-but-real-derived nature (ADR-0003; `data/README_personal.md`).

**Step 2 — subset builder** (`ml/ingest/subset.py`). Memory-safe two-pass streaming;
samples whole users, preserves all takeover users. First run exposed a sentinel user
with 14M logins (98.6% of rows) → added `--max-user-logins` cap to drop non-human
accounts (ADR-0005). Result: 50k users → **202,284 rows**, mean ~4 logins/user, all
141 takeover rows preserved.

**Step 3 — EDA** (`ml/eda/explore.py`). Full results in
`findings/2026-08-08-phase1-eda.md`. Headlines: takeover label is 0.07% (141 rows
total); `is_attack_ip` 8.7%; 40% of users have a single login; RTT 93.8% missing;
region/city ~42% missing; IP/user-agent near-unique; 391-day clean time span.

**Modelling decision.** Given 141 positives, chose the Freeman likelihood-ratio model
as the primary scorer; `is_attack_ip` reserved for the leakage A/B comparison
(ADR-0004).

**Committed & pushed:** `rba-ml-training@45a5e56` (subset + EDA) to
`github.com/roku-pfi/rba-ml-training`.

**Next:** Step 4 — implement the feature set in `rba-features` + offline replay
driver + parity test.

---

## 2026-08-05 — Plan & architecture direction

**Development plan** created at `plans/development_plan.md`. Initial proposal was a
modular monolith; revised after feedback to a **polyrepo + microservices + Kubernetes**
target with a lean login path (ADR-0001).

**Repos scaffolded:** `rba-features` (shared feature library) and `rba-ml-training`
(offline pipeline), each its own git repo, later pushed to the `roku-pfi` GitHub org.

**Data documentation:** added the dataset acquisition guide and clarified that the
Wiefling dataset is a privacy-preserving synthesis of real data, not rule-invented
synthetic data (ADR-0003).
