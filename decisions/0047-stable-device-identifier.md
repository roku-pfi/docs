# 47. A stable device identifier, scored as a designed term

- Status: Accepted
- Date: 2026-11-03
- Related: RF-02, [ADR-0042](0042-learn-from-completed-logins-and-deployment-population.md)
  (learn only completed logins), [ADR-0043](0043-hybrid-freeman.md) (device class
  in Freeman), [ADR-0045](0045-block-the-source-not-the-account.md) (owner vs.
  attacker failures).

## Context

"Device" meant the class read from the browser: desktop, macOS, Chrome. Every
Mac with Chrome looked the same, so the device was the model's weakest signal
and the owner-vs-attacker split of ADR-0045 rested on the network alone. RF-02
asks for the device itself. The training data has no device ids, so a device
term cannot be learned or validated offline.

## Decision

- **Cookie.** The IdP sets `rokusec_device` on its own origin at every login:
  32 random bytes, `HttpOnly`, `SameSite=Lax`, `Secure` in the cloud, 400 days,
  renewed on use. It never leaves the browser and the IdP.
- **What others see.** The PDP gets `device_id = sha256(tenant_id ":" cookie)`,
  32 hex characters (contracts 0.20.0, `RiskEvaluateRequest.device_id`,
  `LoginEventSnapshot.device_id`). One browser is a different device to each
  tenant, and the stored id cannot be replayed as the cookie.
- **Learning.** The profile keeps the last 20 devices a user *completed* a
  login on (`ProfileState.known_devices`). A failed login, or a step-up not
  passed, teaches no device: an attacker with only the password cannot make
  their browser known.
- **Scoring.** Not a Freeman likelihood: a known device adds a fixed
  −ln 10 ≈ −2.30 to the log-risk (`KNOWN_DEVICE_LLR`), shown as its own reason.
  An unknown or missing device adds nothing (cookies are cleared and devices
  replaced); a new device for a user who has known ones is an informational
  reason (`device_new`). Rules still only raise the action, so impossible
  travel, hosting networks and the guard escalate as before.
- **Failures (ADR-0045).** A wrong password on a known device counts as the
  owner's own, as one from a familiar network does.

## Consequences

- "Known device" now means this laptop, not any Mac. The owner on their own
  laptop from a new network scores lower; an attacker never benefits.
- The term's size is a design choice, not a measured one. It is a setting, and
  the e2e checks the behaviour (Martín's usual login carries it).
- Someone who steals the cookie *and* the password looks like the owner's
  device. Then the network signals and the rules still apply.
- The cookie is a security identifier, strictly needed for the service;
  informe note 21 covers the privacy wording.
