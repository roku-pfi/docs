# ADR-0005: Data cleaning — excluding non-human sentinel accounts

- Status: Accepted
- Date: 2026-08-08

## Context

The first subset build (50k users) produced 14.2M rows — ~70× more than expected.
Diagnosis: a single sentinel user ID (`-4324475583306591935`) has **14,025,899
logins** (98.6% of that subset), appearing with many different IPs and mostly
`Login Successful = False`. It is an aggregation bucket for non-attributable logins,
not a real person. The real per-user distribution is median 2 / mean ~4 logins.

## Decision

Exclude non-human accounts from the modelling subset via a configurable cap:
`ml/ingest/subset.py --max-user-logins` (default 10,000). Any user with more logins
than the cap is dropped. This removes the sentinel (and any bot-like accounts) while
keeping legitimate heavy users (the published max legitimate user had ~5,972 logins).

## Consequences

- After the fix: 50k users → **202,284 rows**, mean ~4 logins/user (matches the
  published distribution), with all 141 takeover rows preserved.
- Must be documented as a preprocessing step in the thesis (with the sentinel id and
  the cap value), as it materially changes the dataset.
- `is_attack_ip` / failed-login analyses that intentionally include the sentinel
  should be run separately on the raw data, not the cleaned modelling subset.
