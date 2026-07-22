# harn-stripe-connector

Pure-Harn [Stripe](https://stripe.com/) connector for the Harn orchestrator.
Verifies signed inbound billing webhooks, normalizes Stripe `checkout.*`,
`invoice.*`, and `customer.subscription.*` events into a canonical envelope, and
dispatches typed outbound REST calls for customers, checkout, the billing
portal, subscriptions, products, prices, meters, and meter events.

This is a first-party **inbound + outbound** connector package implementing
Harn Connector Contract v1.

## User story

Receive a signed billing webhook (checkout completed, invoice failed,
subscription changed), map it to the customer and subscription, then let an
agent react — provision access, dunning, or a portal link — and drive metered
billing through Stripe's v2 meter events.

## Install

No release tag is published yet. Use the `main` branch for package-manager
smoke tests:

```sh
harn add github.com/burin-labs/harn-stripe-connector@main
```

Use a path checkout for local multi-repo development:

```toml
[dependencies]
harn-stripe-connector = { path = "../harn-stripe-connector" }
```

## Setup

Store two secrets before enabling Stripe bindings:

| Secret                  | Where to find it                                                                                       |
| ----------------------- | ------------------------------------------------------------------------------------------------------ |
| `stripe/api-key`        | Stripe Dashboard → Developers → API keys → **Secret key** (`sk_test_...` / `sk_live_...`)               |
| `stripe/webhook-secret` | Stripe Dashboard → Developers → Webhooks → your endpoint → **Signing secret** (`whsec_...`, shown once) |

Then connect:

```sh
harn connect stripe
harn connect status --connector stripe --json
```

## Usage

### Webhook trigger: react to a completed checkout

```harn
import stripe_connector from "harn-stripe-connector"

trigger billing_events on stripe {
  source = {
    kind: "webhook",
    events: ["checkout.session.completed", "invoice.payment_failed"],
  }
  on event {
    if event.kind == "checkout.completed" {
      // event.payload has flat customer / subscription / client_reference_id /
      // metadata, plus the full upstream event under `raw`.
      provision_access(event.payload.client_reference_id, event.payload.subscription)
    }
    if event.kind == "invoice.payment_failed" {
      start_dunning(event.payload.customer)
    }
  }
}
```

Normalized event kinds:

| Stripe event type                         | Normalized kind               |
| ----------------------------------------- | ----------------------------- |
| `checkout.session.completed`              | `checkout.completed`          |
| `invoice.paid`                            | `invoice.paid`                |
| `invoice.payment_failed`                  | `invoice.payment_failed`      |
| `customer.subscription.created` (etc.)    | `subscription.created` (etc.) |
| `customer.subscription.trial_will_end`    | `subscription.trial_will_end` |
| any other verified event                  | `event`                       |

The dedupe key is `stripe:<event_id>`.

### Report metered usage (v2)

```harn
stripe_connector.call("meter_event.create", {
  event_name: "api_request",
  stripe_customer_id: event.payload.customer,
  value: 5,               // v2 sends value as a string on the wire
  identifier: "req_" + request_id,   // idempotency for the meter event
  api_key: env("STRIPE_API_KEY"),
})
```

### Create a subscription checkout session

```harn
stripe_connector.call("checkout_session.create", {
  mode: "subscription",
  line_items: [{price: "price_pro_monthly", quantity: 1}, {price: "price_metered"}],
  success_url: "https://app.example.com/welcome",
  cancel_url: "https://app.example.com/pricing",
  client_reference_id: "user_42",
  subscription_metadata: {team: "acme"},
  allow_promotion_codes: true,
  idempotency_key: "checkout_user_42",
  api_key: env("STRIPE_API_KEY"),
})
```

## Normalized event envelope

`normalize_inbound` returns the canonical tagged `NormalizeResult`
(`{type: "event" | "reject", ...}`). The event payload is:

```text
{
  provider: "stripe",
  event_id,               // Stripe evt_... id
  event_type,             // raw Stripe event type
  livemode,
  customer,               // customer id
  subscription,           // subscription id (data.object.id for subscription.* events)
  status,                 // data.object.status
  client_reference_id,    // present on checkout sessions
  metadata,               // data.object.metadata
  raw,                    // the full upstream Stripe event
}
```

## Webhook verification

Stripe signs webhooks with HMAC-SHA256. The
`Stripe-Signature: t=<unix>,v1=<hex>[,v1=<hex>...]` header carries the HMAC of
the literal string `"<t>.<raw_body>"`, keyed by the endpoint **signing secret**
(`whsec_...`). The connector recomputes the HMAC, accepts when **any** `v1`
entry matches in constant time, and — because the timestamp is part of the
signed message — also enforces a 5-minute freshness window to defeat replays.
When **no** signing secret is configured, inbound is rejected (fail closed).

## Outbound methods

| Method                          | HTTP                                                        | Approval |
| ------------------------------- | ---------------------------------------------------------- | -------- |
| `customer.get`                  | `GET /v1/customers/{id}`                                    | no       |
| `customer.create`               | `POST /v1/customers`                                        | yes      |
| `checkout_session.create`       | `POST /v1/checkout/sessions`                                | yes      |
| `billing_portal_session.create` | `POST /v1/billing_portal/sessions`                          | yes      |
| `subscription.get`              | `GET /v1/subscriptions/{id}`                                | no       |
| `subscription.update`           | `POST /v1/subscriptions/{id}` (body passthrough)           | yes      |
| `subscription.cancel`           | `DELETE /v1/subscriptions/{id}` or `cancel_at_period_end`  | yes      |
| `meter_event.create`            | `POST /v2/billing/meter_events` (JSON)                     | yes      |
| `meter.list` / `meter.create`   | `GET` / `POST /v1/billing/meters`                          | list: no |
| `product.list` / `product.create` | `GET` / `POST /v1/products`                              | list: no |
| `price.list` / `price.create`   | `GET` / `POST /v1/prices`                                  | list: no |
| `webhook_endpoint.list` / `.create` | `GET` / `POST /v1/webhook_endpoints`                   | list: no |
| `test_clock.create` / `.advance` | `POST /v1/test_helpers/test_clocks[/{id}/advance]`        | yes      |
| `api.request`                   | raw `{method, path, body, api_version}` escape hatch       | no       |

`methods()` exposes the `requires_approval` flag per method; the orchestrator
gates mutating calls behind its approval boundary. v1 bodies are
form-encoded; v2 (`meter_event.create`) and `api.request` against a `/v2` path
send JSON. Override the API host per call with `api_base_url` or via
`STRIPE_API_BASE_URL` (used by the mock-HTTP tests).

## Verification

```sh
harn check src
harn lint src
harn fmt --check src tests
for test in tests/*.harn; do harn run "$test"; done
harn connector check .
```

`harn connector check .` runs the contract fixtures in `harn.toml`, which cover
verified `checkout.session.completed`, `invoice.payment_failed`, and
`customer.subscription.deleted` events plus stale-signature and
wrong-signature rejections. The package-level gate is `harn connector test .`.

## License

Dual-licensed under MIT and Apache-2.0.

- [LICENSE-MIT](./LICENSE-MIT)
- [LICENSE-APACHE](./LICENSE-APACHE)
