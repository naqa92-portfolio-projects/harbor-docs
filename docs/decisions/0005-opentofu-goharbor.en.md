# 0005 — Configure Harbor with OpenTofu rather than harbor-cli

**In one sentence**: Harbor's configuration (projects, replications, robots, retention…) is OpenTofu code with the official provider, because harbor-cli printed secrets and was not idempotent.

| Status | Date | Scope |
|---|---|---|
| Accepted (supersedes the initial choice of harbor-cli) | September 2026 | Projects, proxy caches, replications, robots, immutability, retention, webhooks, deployment security |

## Context

Harbor itself is deployed by Argo CD from its Helm chart. But its configuration lives in its database and API. It must be **declarative**, **idempotent** (a second run changes nothing) and never write a secret to Git, to logs or to plain state.

What evaluating harbor-cli (pre-1.0) found:

- no working `--password-stdin` when piped: the admin password ends up in the process arguments or in an interactive prompt;
- `robot create` **prints the robot secret on standard output**;
- missing features: no scheduled retention, no non-interactive immutability, no namespace flattening in replication;
- imperative commands: idempotency would have to be scripted around list/compare/create.

The Crossplane providers for Harbor are abandoned or single-maintainer, and a Crossplane control plane costs memory on a single node.

## Decision

- The configuration is OpenTofu code in `tofu/harbor`, with the official `goharbor/harbor` provider, run by `task harbor:configure`. `task harbor:plan` only exits 0 when nothing would change.
- Robot secrets are generated in OpenBao and passed **write-only** (`secret_wo`, never in state).
- The state stays local, ignored by Git, and **encrypted** with the OpenBao Transit key provider.
- Replication filters name their repositories explicitly, because Harbor's `github-ghcr` adapter cannot list GHCR. The golden filter is derived from the catalog.

## Alternatives set aside

- **harbor-cli**: leaks secrets, not idempotent, incomplete.
- **Crossplane**: unmaintained Harbor providers, memory cost.
- **curl scripts against the API**: rewriting what the provider already does.

## Trade-offs and cost

- The `dhi.io` proxy cache credential and the webhook token have no write-only attribute: they are the secrets present in the encrypted state.
- The provider does not detect every manual change (a replication filter edited in the UI does not show as drift; `-replace` is needed). See [What broke](../incidents.md#the-harbor-provider-does-not-see-filter-drift).
- Rotating a robot secret requires bumping `secret_wo_version`.

## What would make me change my mind

A harbor-cli 1.0 fixing secret handling would not be enough: the deciding argument remains being declarative.

## In the code

- [`tofu/harbor/main.tf`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/tofu/harbor/main.tf)
- [`Taskfile.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/Taskfile.yml)
