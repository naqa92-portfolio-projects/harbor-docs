# 0008 — Grype and Trivy diverge: Dependency-Track is authoritative

**In one sentence**: three scanners look at the same images and give different CVE lists; we do not reconcile them and never add them up, we triage in Dependency-Track.

| Status | Date | Scope |
|---|---|---|
| Accepted | September 2026 | Which vulnerability counts are authoritative |

## Context

Three scanners look at the same images:

- **Trivy in CI** (`build-image.yml`), enforcing, with the base image's VEX;
- **Trivy in Harbor** (scan on push, deployment security);
- **Grype in the cluster** (Kubescape), on running images.

Trivy and Grype share neither vulnerability databases, nor matching algorithms, nor the way they apply VEX. Their lists diverge, sometimes on severity, sometimes on whether a CVE is present at all. Dependency-Track matches the attested SBOM against its own sources.

## Decision

- Trivy remains the build gate (CI) and the registry gate (Harbor).
- Grype remains Kubescape's own view of running images; it is not reconciled with Trivy.
- Dependency-Track, fed by `dt-bridge`, is the triage source of truth. The *image posture* dashboard labels each CVE count with its source (`harbor-trivy`, `dependency-track`), before and after VEX.

## Alternatives set aside

- **A single scanner everywhere**: Kubescape embeds Grype, and Trivy is Harbor's native scanner; replacing either would lose an integration.
- **Adding up or merging counts**: we would count the same flaw twice, or hide a useful disagreement.

## Trade-offs and cost

- A CVE can block a build in Trivy and be absent from Grype, or the reverse. Each tool is read on its own terms, and triage happens in Dependency-Track.
- The dashboard never adds up the counts of two scanners.

## In the code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`platform/observability/grafana/image-posture.json`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/observability/grafana/image-posture.json)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
