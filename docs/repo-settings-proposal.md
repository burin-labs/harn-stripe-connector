# Repository settings proposal

This page proposes settings changes for `burin-labs/harn-stripe-connector`. Nothing here has
been applied. Repository and organization settings are the founder's to change,
so this file is a request, not a record.

Status: **proposed, not applied** (2026-09-01).

## Proposals

### 1. Keep issues enabled, keep discussions off

Issues stay on so internal automation and the fleet can file against this
repository. Discussions have never been enabled and there is no audience for
them, so leave them off.

Verification before acting: none needed, this proposal changes nothing.

### 2. Restrict who can open a pull request from a fork

External pull requests are closed unread per `CONTRIBUTING.md`. Requiring
organization membership to open a pull request against `main` removes the
review burden and stops an outside contributor from spending effort on a change
that will not land.

Verification before acting: confirm no non-member has an open pull request
here, so nothing in flight is cut off.

### 3. Point the repository description and homepage at the Harn guide

A visitor arriving from the package index should reach the connector authoring
guide, not guess from the source tree.

Verification before acting: confirm the target guide URL resolves.

## Not proposed

- Archiving. The package is still built and released by the fleet.
- Disabling issues. Internal automation files here.
- Any branch-protection change. Those are set org-wide and are out of scope for
  a hygiene sweep.
