# Changelog

All notable changes to `harn-stripe-connector` will be documented in
this file. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [Unreleased]

### Fixed

- Pin the required Stripe API version on v2 meter-event requests, with an
  explicit per-call override for deliberate API upgrades.

### Added

- Initial pure-Harn Stripe connector implementing the connector interface
  (`provider_id`, `kinds`, `methods`, `payload_schema`, lifecycle,
  `normalize_inbound`, `call`).
- Signed webhook verification: HMAC-SHA256 over `"<t>.<raw_body>"` keyed by the
  endpoint signing secret, accepting any matching `v1` entry in constant time,
  plus a 5-minute freshness window for replay protection. Inbound fails closed
  when no signing secret is configured.
- Normalization of `checkout.*`, `invoice.*`, and `customer.subscription.*`
  events into a canonical billing envelope (`provider`, `event_id`,
  `event_type`, `livemode`, `customer`, `subscription`, `status`,
  `client_reference_id`, `metadata`, `raw`). `checkout.session.completed` maps
  to kind `checkout.completed`, `customer.subscription.<verb>` to
  `subscription.<verb>`, and any other verified event to the generic `event`
  kind rather than being rejected. Dedup is on the Stripe event id
  (`stripe:<evt_id>`).
- Outbound `call(method, args)` dispatch for customers, checkout sessions,
  billing portal sessions, subscriptions (get / update / cancel), meters, meter
  events (v2 JSON), products, prices, webhook endpoints, and test clocks, plus a
  raw `api.request` escape hatch. Mutating methods are flagged
  `requires_approval` via `methods()`.
- A deterministic rails-style form encoder for v1 bodies (nested dicts and
  lists via bracket keys, booleans as `true`/`false`, nil fields omitted),
  JSON bodies for v2 meter events with `value` as a string, `Idempotency-Key`
  threading, HTTPS-only outbound with SSRF host guards, and a single 429/503
  retry honoring a small `Retry-After`.
- Signed-fixture smoke tests covering normalization mapping, signature
  pass/fail (tampered, wrong secret, missing secret, missing signature),
  stale-timestamp rejection, multiple-`v1` acceptance, unknown-event
  passthrough, the dedup key, exact form-encoder output, and a mock-HTTP call
  smoke across every method.

[Unreleased]: https://github.com/burin-labs/harn-stripe-connector/compare/main
