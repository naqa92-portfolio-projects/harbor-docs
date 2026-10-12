# 0006 — Transparent registry mirror and per-namespace trust tiers

**In one sentence**: every pull goes through Harbor, but with automatic fallback to upstream, so Harbor is never a single point of failure; and admission is not imposed on the components that run it.

| Status | Date | Scope |
|---|---|---|
| Accepted | September 2026 | How nodes pull images, and what admission requires per namespace |

## Context

We want every pull to go through Harbor (cache, scanning, a single egress point). But Harbor, its database, the CNI, the GitOps controller and the policy engine run on the same cluster. If they could only pull their images through Harbor, **a Harbor outage would prevent Harbor from restarting**.

Likewise, enforcing the full chain of trust on the components that implement admission would create a dependency loop.

## Decision

- **Transparent mirror**: containerd on the node has `hosts.toml` mirrors for `docker.io`, `quay.io`, `ghcr.io`, `registry.k8s.io` and `dhi.io`, pointing to the matching Harbor cache project, **with upstream as fallback**. Manifests keep their original references; if Harbor is down, the pull goes upstream. Bootstrap needs no special case.
- **Trust tiers** by namespace label:
    - `workload`: full chain, enforced;
    - `platform`: registry allow-list and vendor signatures, in `Audit` mode (reported, not blocked);
    - no tier: every pod denied, so a forgotten label never turns the checks off;
    - bootstrap (`kube-system`, `cilium`, `argocd`, `kyverno`): excluded by name, see the [threat model](../modele-de-menace.md#bootstrap-namespaces-an-accepted-exclusion).

## Alternatives set aside

- **Rewriting every reference to Harbor** (`harbor…/dockerhub-proxy/…`): Harbor becomes a single point of failure for its own restart.
- **The same requirements everywhere**: a dependency loop, and a cluster that can no longer converge after an outage.

## Trade-offs and cost

- Workloads must reference Harbor `golden`/`apps` explicitly: a `docker.io` reference served by the mirror is still denied, because admission sees the reference, not the actual pull path.
- Platform components keep their upstream references and vendor signatures; their deviations are visible without blocking recovery.
- The bootstrap exclusion is an accepted risk, limited to cluster administrators.
- Cache fill is asynchronous and can trim multi-arch indexes ([ADR 0001](0001-format-signature-replication.md)).

## What would make me change my mind

A production cluster with a highly available registry outside the cluster: going through the registry could then be required for everyone.

## In the code

- [`scripts/platform-up.sh`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/scripts/platform-up.sh)
- [`platform/kind/cluster.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/kind/cluster.yaml)
- [`policies/tiers`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/policies/tiers)
- [`platform/config/tiers/namespaces.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/config/tiers/namespaces.yaml)
