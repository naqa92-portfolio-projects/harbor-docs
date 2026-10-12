# 0007 — Keyless signing with Sigstore, identity bound to this repository's `main` branch

**In one sentence**: images are signed *keyless* with the GitHub workflow's OIDC identity, and only this repository's `main` branch produces images admissible in production.

| Status | Date | Scope |
|---|---|---|
| Accepted, narrowed on 2026-09-28 | September 2026 | Signatures and attestations of golden, application and test images |

## Context

Admission must prove that an image was built by the platform's workflow, from this repository. Key-based signing assumes a private key stored, rotated and protected somewhere (a KMS or HSM in a bank); a public portfolio repository has none of that.

Sigstore keyless signing binds a short-lived Fulcio certificate to the workflow's GitHub Actions OIDC identity, and records every signature in the Rekor transparency log.

## Decision

- `build-image.yml` signs and attests keyless with cosign v3: signature, CycloneDX SBOM, SPDX SBOM and SLSA v1 provenance.
- Admission checks the issuer `https://token.actions.githubusercontent.com` and the identity `build-image.yml@refs/heads/main` **of this repository only**.
- **Repository binding**: `build-image.yml` is a public reusable workflow; without a check, its signing identity is the same whichever repository calls it. The build job therefore only runs if `github.repository` is this repository, admission requires the verified provenance to name this repository as the caller, and `dt-bridge` requires it in the certificate extension.
- **Signing only the digest just built**: the signing steps only sign the digest this same job has just built, never a pre-existing tag on GHCR.
- **Pinned DHI key**: the DHI base image signature is verified against `.github/keys/dhi.pub`, committed to the repository, not against a key downloaded at build time.
- **Environment-specific trust**: a platform deployed from a working branch trusts that one branch in addition to `main`, and only in that environment (see [threat model](../modele-de-menace.md#environment-specific-trust)).

## Why the decision was narrowed

The first version trusted `main` **or any `prd-*` branch**. Review showed that any `prd-*` branch could then sign images admissible in production, which is broader than intended. Since 2026-09-28: `main` only by default, plus repository binding, signing only the digest just built, and the pinned DHI key.

## Alternatives set aside

- **A classic cosign key in a GitHub secret**: a long-lived key to protect and rotate, for no gain here.
- **A key in a KMS**: the right answer in a bank, but there is no KMS in a portfolio. Switching only changes the attestor definition in the policies.

## Trade-offs and cost

- No private key to manage; every signature is public and auditable in Rekor.
- Signing and verification depend on the public Sigstore infrastructure (Fulcio, Rekor, TUF root).
- **Write access to `main` is part of the chain of trust.** The ruleset blocks deletion and force-pushes, not who can push. Here, that is the sole owner.
- After each working branch merge, pinned images must be re-pinned (`release-repin.yml`).

## What would make me change my mind

A regulated context: a KMS/HSM key or a private Sigstore, required reviews and restricted pushes to `main`.

## In the code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`policies/workload/image-signature.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/policies/workload/image-signature.yaml)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
- [`.github/keys/dhi.pub`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/keys/dhi.pub)
- [`.github/workflows/release-repin.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/release-repin.yml)
