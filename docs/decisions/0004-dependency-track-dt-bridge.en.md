# 0004 — Triage vulnerabilities in Dependency-Track, fed by an in-house bridge

**In one sentence**: Dependency-Track is the source of truth for CVE triage, fed with the SBOM signed at build time by `dt-bridge`, a small Python service filling what Dependency-Track cannot do on its own.

| Status | Date | Scope |
|---|---|---|
| Accepted | September 2026 | SBOM storage, vulnerability triage, VEX handling |

## Context

Every image carries a CycloneDX SBOM attested at build time, and its golden base comes with vendor OpenVEX statements (Docker Hardened Images publishes `not_affected` statements). Kubescape produces runtime VEX.

Triage needs one place to see flaws per image version and apply VEX, **fed with the attested SBOM, not a regenerated one**. But Dependency-Track 5.1 imports CycloneDX SBOMs and VEX, **not OpenVEX** (DependencyTrack/dependency-track#7094), and has no Harbor integration.

## Decision

- Dependency-Track 5.1, on CloudNativePG, is the triage source of truth.
- `dt-bridge`, a FastAPI service itself built through the golden path, connects the two:
    - on every Harbor webhook, it reads the image's CycloneDX attestation, **verifies its signature against the build identity**, and uploads the SBOM unchanged to project `<harbor project>/<repository>`, version = tag. If the provenance fails verification, nothing is uploaded;
    - it reads the DHI base from the verified provenance, fetches its OpenVEX, converts the `not_affected` statements into CycloneDX VEX and uploads it once Dependency-Track has analysed the SBOM;
    - it forwards Kubescape's runtime VEX the same way;
    - it exports the posture metrics of the Grafana dashboard.
- All the OpenVEX vocabulary is isolated in one module (`vex.py`), removable in one piece the day Dependency-Track imports OpenVEX.

## Alternatives set aside

- **Letting Harbor triage alone**: Harbor scans, but handles neither the triage lifecycle nor per-component VEX.
- **Regenerating an SBOM on the Dependency-Track side**: we would triage something other than what was signed at build time.

## Trade-offs and cost

- The SBOMs in Dependency-Track are exactly those attested and signed at build time: that is the main benefit.
- **Upstream limit**: Dependency-Track matches Debian advisories by *source* package name, while the SBOM lists *binary* packages (DependencyTrack/dependency-track#6132, #6957). No Debian flaw therefore shows up for golden images, and the effect of a DHI VEX on each individual flaw is not observable yet. The VEX upload itself is accepted and processed (`COMPLETED`).
- `dt-bridge` is one more component to maintain, with its own tests (26 pytest tests).

## What would make me change my mind

Dependency-Track importing OpenVEX natively: `vex.py` goes away.

## In the code

- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
- [`platform/workloads/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/platform/workloads/dt-bridge)
- [`platform/apps/templates/dependency-track.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/dependency-track.yaml)
