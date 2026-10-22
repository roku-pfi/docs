# 41. How the presenter panel reaches the attackers and the platform

- Status: Accepted
- Date: 2026-10-22
- Related: [ADR-0035](0035-demo-around-personas-and-tenants.md) (decides that
  attackers are Lambdas and that a presenter panel drives them),
  [ADR-0032](0032-real-login-signals.md) (simulation tickets).

## Context

ADR-0035 decided *what* the stage machinery is: three attacker Lambdas in other
regions and a presenter panel with attacker scenes, simulated travel, load and
reset. It left open how the panel calls the Lambdas, where the panel runs, and
what "reset" touches.

## Decision

- **Attackers are reached through public function URLs with a shared secret**
  (`X-RokuSec-Attacker-Secret`); without it they answer 404. Invoking them
  through the Lambda API would need an IAM identity for a pod (IRSA or Pod
  Identity) in the throwaway runtime stack, for three functions that only ever
  log in to our own IdP. The secret lives in the permanent stack's state and
  reaches the bank through `aws-up.sh`'s session values, like the CloudFront
  origin secret.
- **The panel stays inside the bank** at `/presenter` (replacing `/walkthrough`),
  as a separate module that the bank's own pages never link to. It is protected
  by `PRESENTER_SECRET`. In the cloud it does not exist without one; locally it
  is open. A separate deployable would add an image, a release and an ingress
  for one page.
- **Load calls the PDP directly** from inside the cluster (`/risk/evaluate`),
  as the k3d HPA test already did. What it shows is the PDP's latency and
  scaling, which is the RNF-07 claim; going through the IdP's login would mean
  sessions and challenges for fake users in Diego's event list.
- **Reset touches only the profile store.** "Limpiar fallos" empties
  `failed_login_ts` in Martín's profiles (both tenants) and keeps his history;
  "Olvidar perfil" deletes them, so his next login is a first login.
- **The cloud no longer seeds the TEST-NET home history.** It never matched a
  real login (Martín's first phone login scored CRITICAL against it). Real
  logins build the profile; locally the seed stays, because every local request
  comes from a private address.
- Locally, without Lambdas, the attackers are **stand-ins** that post the
  region's country and Amazon's ASN in the login body. Only a local IdP believes
  that; the panel labels them "atacantes locales".

## Consequences

- The function URLs are reachable by anyone who has the secret. They can only
  send up to 20 logins per call to our IdP, and the account's Lambda
  concurrency caps the rest.
- The bank image carries `redis` and `python-multipart` for the panel.
- In the cloud, the Roku Bank scenes start from no history: Martín's first
  logins are challenged until his profile builds up. The run sheet (D-4) has to
  start with a few ordinary logins in monitor mode.
