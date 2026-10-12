# 0002 — Write every admission policy in Kyverno CEL

**In one sentence**: Kyverno's legacy policies (`ClusterPolicy`) are removed in 1.20, so every policy is written in the new CEL types, aligned with native Kubernetes admission.

| Status | Date | Scope |
|---|---|---|
| Accepted | September 2026 | Every policy under `policies/` |

## Context

Admission must enforce the image chain of trust (signature and attestations from the build workflow, a catalog base read from the provenance, labels, Harbor as the only registry) and Pod Security `restricted` in workload namespaces. It must report deviations in platform namespaces, and be **testable offline in CI**.

Kyverno 1.19 offers two families: the legacy `ClusterPolicy`/`Policy` (deprecated, removed in 1.20) and the CEL types (`ValidatingPolicy`, `ImageValidatingPolicy`, `MutatingPolicy`), aligned with Kubernetes' `ValidatingAdmissionPolicy`. The other candidate was OPA Gatekeeper with Rego.

!!! tip "CEL in a nutshell"
    *Common Expression Language*: the small expression language Kubernetes natively uses for its admission rules. A CEL policy reads like a boolean condition on the submitted object.

## Decision

- Every policy uses a CEL type. No `ClusterPolicy` or `Policy` in the repository.
    - Workload (enforced): `workload-image-signature`, `workload-golden-base`, `workload-registry`, `workload-image-labels`.
    - Workload (warning): `workload-golden-base-deprecated`.
    - Workload (mutation): `workload-pod-security`, which labels workload namespaces Pod Security `restricted`.
    - Trust tiers: `tier-label` (only admins set the tier) and `tier-required` (any pod outside a valid tier is denied).
    - Platform (audit): `platform-registry-allow-list`, `platform-vendor-signatures`.
- Pod Security `restricted` is the **native Pod Security Admission**, not re-implemented as a policy.
- Policies read their parameters from generated ConfigMaps (golden catalog, allowed registries): a catalog change needs no policy change.
- Admitted images are rewritten to their digest (`mutateDigest`).
- Enforcing policies run with `failurePolicy: Fail`: if Kyverno does not answer, the pod is denied.
- Tests: `kyverno test` (one suite per policy) and Chainsaw on an ephemeral kind cluster, on every pull request.

## Alternatives set aside

- **Legacy `ClusterPolicy`**: removed in Kyverno 1.20, it would be debt from day one.
- **OPA Gatekeeper + Rego**: very common in banks, but signature verification needs an external data provider (Ratify) and Rego is one more language to master. A side-by-side comparison stays on the [roadmap](../limites.md#roadmap).

## Trade-offs and cost

- CEL policies report `fail`, never `warn`: the "deprecated base" case shows as `fail` in PolicyReports while admission only shows a warning.
- A denial by a CEL policy emits no Kubernetes event: violations are counted from the API server audit log.
- `kyverno test` deadlocks when two image-verifying policies evaluate the same resource (upstream bug kyverno/sdk#125), hence one test suite per policy.

## What would make me change my mind

A client already running Gatekeeper everywhere: I would run the planned comparison before imposing a second engine.

## In the code

- [`policies`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/policies)
- [`tests/kyverno`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/tests/kyverno)
- [`tests/chainsaw`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/tests/chainsaw)
- [`.github/workflows/policies.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/policies.yml)
