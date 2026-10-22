# 32. Real login signals: client IP behind the edge, offline IP geolocation

- Status: Accepted
- Date: 2026-10-21
- Related: [ADR-0022](0022-product-demo-over-gitops.md) (country on the login
  path, travel rule), [ADR-0031](0031-aws-eks-managed-data-plane.md).
- Requirements: **RF-02** (location part), **RNF-02**.

## Context

The IdP never geolocates a real address. `rba_idp/geo.py` maps three RFC 5737
documentation ranges to AR / JP / DE, accepts a presenter override, and treats
every private address as the demo "home". That was right for a laptop: every
request came from `127.0.0.1` or a cluster peer.

In the cloud it is wrong in three ways:

1. The address the IdP sees is the ALB's, not the user's. The user's address is
   in `X-Forwarded-For` / `CloudFront-Viewer-Address`.
2. A real address maps to nothing, so `country` and `asn` are missing for every
   real login and the travel rule and ASN familiarity never fire.
3. `lookup_ip` ignores IPv6. Phones on mobile data are frequently IPv6-only,
   so the "new device / new network" scenario would silently lose its country.

## Decision

### 1. Client IP from trusted proxy headers only

The IdP takes the client address from the forwarding headers **only when the
direct peer is a trusted proxy** (the ALB's subnets in the cloud; nothing
locally). It takes the right-most address that is not a trusted proxy, which a
client cannot forge by sending its own `X-Forwarded-For`. Configured by
`TRUSTED_PROXY_CIDRS`; empty means "use the socket peer" (today's behaviour).

### 2. Offline IP → country / ASN database in the image

**DB-IP Lite** (country + ASN, IPv4 and IPv6, CC BY 4.0, no account needed),
in MMDB format, downloaded at image build time and read with `maxminddb`. The
same code path runs on k3d, in tests and on EKS, and a lookup is an in-memory
read with no network call inside the login latency budget (RNF-07). The
attribution goes in the informe and in the console footer.

CloudFront's `CloudFront-Viewer-Country` / `-ASN` headers were considered: free
and zero maintenance, but they exist only behind CloudFront, so local runs and
tests would need a second code path.

### 3. What stays

- The **presenter override** (`country` / `asn` query parameters) stays, for the
  one scenario that cannot be produced honestly from the cloud: impossible
  travel. Every decision taken with an override is labelled **simulated** in the
  console (a reason flag), so a simulated attempt is never mistaken for a real
  one.
- The TEST-NET mappings stay for unit tests.
- The **hosting/VPN ASN list is unchanged.** Logins from AWS (attacker Lambdas,
  ADR-0035) are classified as hosting networks, which is the correct reading of
  a login from a datacenter. No "lab mode" removes AWS from the list.
- Private-address logins still score as home locally, so the k3d walkthrough
  behaves as before.

## Consequences

- A visitor's real country and ISP reach the PDP, so the "new network" scenario
  is real on any phone.
- IPv6 logins carry signals for the first time.
- The image grows by the database (~10 MB for country + ASN).
- The database ages; it is refreshed whenever images are rebuilt. A stale entry
  degrades to "missing", which the features already treat as not-seen and never
  as a match.
- Impossible travel from a real origin needs a person on a residential or mobile
  network abroad. Planned later as an asynchronous demo, not in this phase.
