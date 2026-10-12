# 0001 — Sign in Sigstore bundle format and replicate referrers into Harbor

**In one sentence**: cosign's modern signature format (Sigstore bundles) survives GHCR → Harbor replication and is read by Kyverno, as long as the `sha256-*` tags carrying the signatures are replicated too.

| Status | Date | Scope |
|---|---|---|
| Accepted (point 4 superseded, see below) | September 2026, M0 spike | Every image signed in GitHub Actions, replicated into Harbor and admitted by Kyverno |

## Context

Images are built in GitHub Actions, signed and attested with cosign, pushed to GHCR, replicated into Harbor, then admitted by Kyverno on their Harbor reference. There are two ways to store a signature:

- cosign's legacy format, `sha256-<digest>.sig` and `.att` tags;
- **Sigstore bundles** attached as OCI *referrers* (artifacts pointing to the image they describe).

Before building anything, a spike had to prove which one survives replication and which one Kyverno can read.

## Decision

1. **Sigstore bundle format** (cosign v3's default). The `.sig`/`.att` tags are deprecated in cosign v3; the platform does not build on them.
2. **The SLSA v1 provenance is written by the workflow itself** and attached with `cosign attest`. It records the base image digest, which the catalog policy then reads. The `actions/attest-build-provenance` action is not used: it stores to GitHub's attestation API and signs with a different identity than the workflow.
3. **Harbor replication copies the referrers fallback tag.** The `golden` and `apps` rules have no tag filter, which replicates the `sha256-*` tags along with the images.
4. ~~Kyverno trusts Harbor's CA through `SSL_CERT_DIR` and a hand-mounted ConfigMap.~~ **Superseded**: **trust-manager** now produces the `kyverno-harborlab-ca` ConfigMap from the local CA; Kyverno mounts it next to the system roots, still through `SSL_CERT_DIR`.

## What the spike measured

| Topic | Observed |
|---|---|
| GHCR storage | GHCR answers 404 on the referrers API. cosign therefore writes a `sha256-<hex>` tag holding an index of the bundles (one signature + one entry per attestation). |
| Replication | Harbor copies the image with an identical digest. With the `{m0,sha256-*}` filter it also copies the signature index and serves it on its own referrers API. **A filter on the image tag alone drops every signature.** |
| Kyverno | `ImageValidatingPolicy` detects the bundle format and reads the signature and the three attestations through Harbor. The unsigned image is denied with a message naming the policy. |
| Kyverno CA trust | The chart's `caCertificates` field **replaces** the certificate store. Adding the whole system bundle to it exceeds the 1 MiB limit of the Helm release Secret, and leaving it out breaks TLS to Sigstore. Hence the mount next to the system roots. |
| Proxy cache fill | Asynchronous. For a multi-arch index pulled for a single platform, Harbor waits up to 20 × 20 s for the other platforms, then pushes **a trimmed index with a new digest**. Measured: about 5 minutes before the tag shows up. |

## Alternatives set aside

- **`.sig`/`.att` tags**: deprecated, bound to disappear.
- **`actions/attest-build-provenance`**: a different signing identity than the workflow, storage outside the registry.
- **The Kyverno chart's `caCertificates`**: replaces the system roots, breaks Sigstore.

## Trade-offs and cost

- Replication rules cannot filter tags finely without losing the signatures.
- The proxy cache can produce digests that differ from upstream for multi-arch indexes: a check by tag must tolerate minutes of delay.
- Registry clients and Kyverno need the platform CA, generated at runtime and never committed.

## What would make me change my mind

GHCR implementing the OCI referrers API: the fallback tag would become useless.

## In the code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`tofu/harbor/main.tf`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/tofu/harbor/main.tf)
- [`platform/apps/templates/kyverno.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/kyverno.yaml)
- [`platform/apps/templates/trust-manager.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/trust-manager.yaml)
- [`spike`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/spike)
