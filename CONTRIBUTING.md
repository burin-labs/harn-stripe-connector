# Contributing

## This repository does not take external contributions

`harn-stripe-connector` is a first-party Burin Labs package. It implements Harn Connector Contract v1 for Stripe: it verifies signed inbound billing webhooks, normalizes `checkout.*`, `invoice.*`, and `customer.subscription.*` events, and dispatches typed outbound REST calls.

Its source, its public contract, and its release cadence are driven by the
Harn runtime and by internal automation that opens most of the pull requests
here. An outside change would land against a contract that moves underneath it
without notice, so pull requests from outside Burin Labs are closed unread.

No release tag is published yet, so there is no stable surface to contribute against.

## Where to send a Stripe request or bug instead

File it against the Harn repository, which owns the connector contract and
decides which providers get built:

- <https://github.com/burin-labs/harn/issues/new>

Label it `area/connectors` and say which Stripe events or operations you need
and what you would do with them. A concrete workflow gets a connector built
sooner than a request for coverage in general.

## If you work at Burin Labs

Install the Harn CLI pinned by `.harn-version`, then run `harn fmt --check src tests`, `harn check .`, `harn test tests --parallel`, and `harn package verify .`.

Run the package gate before you open a pull request. Do not edit anything
between the `<!-- BEGIN HARN SHARED AGENT CONTRACT -->` markers in
`AGENTS.md`, `.github/workflows/ci.yml`, or `.github/dependabot.yml`:
those are projections owned by `harn-bump-fleet`, and a local edit is
overwritten at the next fleet sync.

### Pull request titles

Title every pull request `[Area] Sentence case description`, for example
`[Connector] Reject unsigned webhook deliveries`. Use one of `Connector`,
`Docs`, `CI`, `Tests`, or `Release`. Keep the description to three to
five sentences: what changed, why, the one risk, and how you verified it.
`.github/pull_request_template.md` carries a worked example.

### Labels

`.github/labels.yml` records the label vocabulary. Priority, status, and
effort come from the org taxonomy in
[burin-labs/.github](https://github.com/burin-labs/.github); `area/*` is
local to this repository. Reuse `bug`, `enhancement`, and `documentation`
for type rather than adding a `type/*` prefix.
