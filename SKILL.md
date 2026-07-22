# SKILL: harn-stripe-connector

Trigger recipes and outbound helpers for Stripe via the pure-Harn
`harn-stripe-connector` package. Verify signed billing webhooks, map them to the
customer and subscription, then let an agent react — provision access, dunning,
portal links — and drive metered billing through Stripe's v2 meter events.

## What you get

- **Signed webhook inbound:** HMAC-SHA256 over `"<t>.<raw_body>"` with a
  constant-time compare that accepts any `v1` entry, plus a 5-minute freshness
  window (replay protection). Fails closed when no signing secret is configured.
- **Normalized billing envelope:** `checkout.session.completed` →
  `checkout.completed`, `customer.subscription.<verb>` → `subscription.<verb>`,
  `invoice.paid` / `invoice.payment_failed` pass through, and any other verified
  event maps to the generic `event` kind. Flat `customer`, `subscription`,
  `status`, `client_reference_id`, and `metadata` fields are extracted, with the
  full upstream event kept under `raw`.
- **Typed REST outbound:** customers, checkout sessions, billing portal
  sessions, subscriptions (get / update / cancel), products, prices, meters, and
  v2 meter events, plus test clocks for end-to-end lifecycle runs.
- **Raw `api.request` escape hatch** with automatic form (v1) or JSON (v2) body
  encoding.

## Signature scheme

The `Stripe-Signature: t=<unix>,v1=<hex>[,v1=<hex>...]` header carries the HMAC
of `"<t>.<raw_body>"`, keyed by the endpoint signing secret (`whsec_...`).
Multiple `v1` entries can appear while a signing secret is being rotated; the
connector accepts when any one matches and then enforces the freshness window.

## Trigger recipe: fulfill a completed checkout

```harn
import stripe_connector from "harn-stripe-connector"

trigger fulfillment on stripe {
  source = { kind: "webhook", events: ["checkout.session.completed"] }
  on event {
    // event.payload carries flat customer / subscription / client_reference_id
    // / metadata, with the full upstream event under `raw`.
    provision_access(event.payload.client_reference_id, event.payload.subscription)
  }
}
```

## Trigger recipe: report metered usage (v2, mutating)

```harn
trigger meter_usage on stripe {
  source = { kind: "webhook", events: ["invoice.paid"] }
  on event {
    // v2 meter events send `value` as a string; `identifier` is the idempotency key.
    stripe_connector.call("meter_event.create", {
      event_name: "api_request",
      stripe_customer_id: event.payload.customer,
      value: usage_for(event.payload.customer),
      identifier: event.payload.event_id,
      api_key: env("STRIPE_API_KEY"),
    })
  }
}
```

## Cancel at period end vs immediately

```harn
// Immediate: DELETE /v1/subscriptions/{id}
stripe_connector.call("subscription.cancel", {id: "sub_123", api_key: env("STRIPE_API_KEY")})

// Scheduled: POST cancel_at_period_end=true
stripe_connector.call("subscription.cancel", {
  id: "sub_123",
  at_period_end: true,
  api_key: env("STRIPE_API_KEY"),
})
```

## Required secrets per binding

| Secret                  | Used for                                              |
| ----------------------- | ----------------------------------------------------- |
| `stripe/webhook-secret` | Verify inbound webhooks (signing secret `whsec_...`)  |
| `stripe/api-key`        | Outbound REST as `Authorization: Bearer` (`sk_...`)   |

Restricted keys need write access to Customers, Checkout Sessions, Billing
Portal, Subscriptions, Products, Prices, Billing Meters, and Webhook Endpoints;
metered billing also needs the v2 Billing Meter Events resource.

## Custom API hosts

Set `api_base_url` on each call (or `STRIPE_API_BASE_URL`) to target a
non-default host; the default is `https://api.stripe.com`. v1 bodies are
form-encoded (rails-style brackets); v2 meter events use JSON with `value` as a
string.
