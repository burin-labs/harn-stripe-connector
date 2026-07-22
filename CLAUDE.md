# CLAUDE.md - harn-stripe-connector

Pure-Harn connector package for Stripe (signed billing webhooks + typed REST for
customers, checkout, billing portal, subscriptions, and meter events).

Shared Harn connector authoring rules are in the
[connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md).

Keep this file limited to Stripe-specific notes and local hazards. Put shared connector guidance
in the Harn guide first.

## Provider Notes

- Webhook signing is HMAC-SHA256 over the literal string `"<t>.<raw_body>"` (the unix timestamp, a
  dot, then the raw body), keyed by the endpoint signing secret (`whsec_...`). The header is
  `Stripe-Signature: t=<unix>,v1=<hex>[,v1=<hex>...,v0=...]`. Recompute, accept when **any** `v1`
  entry matches in constant time, then separately enforce a 5-minute freshness window on `t` — the
  timestamp is inside the signed message, so skipping the window loses replay protection. When no
  signing secret is configured, inbound is rejected (fail closed).
- Stripe rotates its signing secrets, and an endpoint can carry more than one active secret during a
  roll, which is why multiple `v1` entries can appear; matching any of them is correct.
- The event `type` lives in the JSON body, not a header. `checkout.session.completed` normalizes to
  kind `checkout.completed`; `customer.subscription.<verb>` normalizes to `subscription.<verb>`;
  `invoice.paid` / `invoice.payment_failed` pass through. Any other verified event normalizes to the
  generic kind `event` rather than being rejected. Dedupe is on the Stripe event id
  (`stripe:<evt_id>`).
- v1 REST (`https://api.stripe.com/v1/...`) takes `application/x-www-form-urlencoded` bodies with
  rails-style bracket nesting: dicts become `parent[key]`, lists become `parent[0]`, booleans
  serialize as `true`/`false`, and nil fields are omitted. Encoding is deterministic
  (alphabetical keys) so request bodies are testable.
- v2 (`https://api.stripe.com/v2/billing/meter_events`) takes a JSON body and requires `value` as a
  **string**. Both v1 and v2 authenticate with `Authorization: Bearer <stripe/api-key>`.
- `Idempotency-Key` is threaded through via the `idempotency_key` arg on mutating calls; a single
  429/503 is retried once (honoring a small `Retry-After`).
- `test_clock.create` / `test_clock.advance` hit the test-mode `/v1/test_helpers/test_clocks` API and
  only work with a test-mode key; use them to drive subscription lifecycle events in end-to-end runs.
- Do not add compatibility shims or deprecation aliases in this nascent package; cut over directly
  when behavior changes.
