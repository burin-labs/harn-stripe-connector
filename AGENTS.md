# AGENTS.md

Pure-Harn connector package for Stripe (signed billing webhooks + typed REST for
customers, checkout, billing portal, subscriptions, and meter events).

Shared connector authoring rules live in the Harn guide:

- [Connector authoring guide](https://github.com/burin-labs/harn/blob/main/docs/src/connectors/authoring.md)

Put shared connector guidance in the Harn guide and keep only
provider-specific notes and local hazards here.

`CLAUDE.md` points here. Edit `AGENTS.md` only.

## Provider notes

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

<!-- BEGIN HARN SHARED AGENT CONTRACT: managed by harn-bump-fleet -->

## Ecosystem working agreement

- Build ambitious outcomes behind small typed interfaces; give behavior one owner
  and generate or parity-test projections instead of duplicating policy.
- Work autonomously within approved scope. Pause for destructive or production effects,
  exceptional spend, material ambiguity, or new authority.
- Treat stop, wait, stand down, pivot, and steer as control events.
- Use the smallest owning product-path check. Add a falsifier for contested, load-bearing,
  or potentially vacuous claims; record controls, recovery, and blind spots.
- Evidence follows source/artifact identity. Reuse proof when relevant code, build inputs,
  and dependencies are unchanged. Repeat affected checks for relevant changes, failures,
  deployment, or packaging differences. Do not rebuild or recapture solely for main.
- Ship means owning-main integration with terminal merge and applicable release/deploy
  checks. Confirm landed content and result; an open PR is incomplete.
- Use `ship` with a deployed Smart Ship caller; otherwise use `gh pr merge --squash --auto`.
  Never use `--admin`; incidents use `bypass-ci`, `bypass-merge-queue`, or `force-merge`.

<!-- END HARN SHARED AGENT CONTRACT -->
