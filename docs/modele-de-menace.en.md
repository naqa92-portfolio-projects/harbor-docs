# Threat model

harbor-lab decides which container images may run on the cluster. This page states **what admission checks**, tier by tier, and **the exceptions** that trust relies on.

## Assets, threats, controls

| What we protect | Threat | Control |
|---|---|---|
| Workload images | Unsigned, tampered or foreign-built image | Policy `workload-image-signature`: keyless signature plus CycloneDX, SPDX and SLSA v1 attestations issued by the `build-image.yml` workflow, **and** a provenance naming this repository as the caller |
| Base images | Image built on an unknown or end-of-life base | Policy `workload-golden-base`: the base digest read from the verified provenance must be `supported` or `deprecated` in the catalog (`deprecated` produces a warning) |
| Registry path | Image pulled from outside the governed Harbor projects | Policy `workload-registry`: only `harbor…/golden/` and `harbor…/apps/` |
| Traceability | Image without source, revision or owner | Policy `workload-image-labels`: required OCI and `io.harborlab.*` labels |
| Node | Privileged or root container | Native Pod Security Admission, `restricted` profile enforced on every workload namespace |
| Tag drift | A tag re-pointed to another image after admission | Admission rewrites every admitted image to its digest (`mutateDigest`) |

## Trust tiers

| Tier | Namespaces | Admission |
|---|---|---|
| Workload | `harborlab.io/tier=workload` | Full chain, enforced (`Deny`) |
| Platform | `harborlab.io/tier=platform` | Reported only (`Audit`): allowed registry, vendor signatures for Kyverno, Cilium and Argo CD |
| Bootstrap | `kube-system`, `cilium`, `argocd`, `kyverno` | None: excluded by name |
| None | any other namespace | Every pod denied (`tier-required`): enforcement fails closed |

Only platform administrators (cluster admins and Argo CD, which syncs labels from Git) choose a tier. Someone allowed to edit a namespace can put it in `workload`, but can neither remove the label, change it, nor choose `platform` (policy `tier-label`).

!!! note "An honest limit"
    The vendor-signature rules for Cilium and Argo CD only apply when those images run in a `platform` namespace. Here they only run in the excluded `cilium` and `argocd` namespaces. In practice, only Kyverno's images (Policy Reporter, for instance) meet this rule.

## Bootstrap namespaces: an accepted exclusion

`kube-system`, `cilium`, `argocd` and `kyverno` are excluded by name from the policies, even if they carry a label. They run what admission itself depends on: CoreDNS and the API server add-ons, the CNI, the GitOps controller that deploys the policies, and the policy engine.

Enforcing policies on them creates a **dependency loop**: a Kyverno, Rekor or registry outage, or a policy mistake, would block the pods needed to recover, and a cluster restart could never converge.

This is a documented, accepted risk:

- Someone able to create pods in these namespaces bypasses admission. Access is limited to cluster administrators and the controllers installed there; no workload runs in them.
- Their images come from Helm charts and values versioned and reviewed in Git, not from runtime choices.

## Keyless signing on public Rekor

Images are signed **without a key**: GitHub Actions obtains a short-lived Fulcio certificate bound to the workflow identity (`build-image.yml` on this repository's `main` branch), and every signature is recorded in the public Rekor transparency log. No private key to steal or rotate, and every signature is auditable.

The trade-offs, for a regulated environment such as a bank:

- Signing depends on the public Sigstore infrastructure (Fulcio, Rekor, TUF root), reachable from CI and from admission.
- The log publishes the identity, the repository and the digest: fine for a public repository, not for confidential build metadata.
- Trust rests on GitHub's OIDC issuer and on **write access** to the repository. The `main` ruleset blocks deletion and force-pushes, but does not restrict who can push: the signing identity is therefore bound to whoever has that access (here, the sole owner).

The banking alternative is key-based signing with a key held in a KMS or HSM (`cosign --key <kms-uri>` supports AWS KMS, GCP KMS, Azure Key Vault and Vault / OpenBao Transit), possibly with a private Sigstore instance. Only the attestor definition in the policies changes.

## Environment-specific trust

The committed policies trust `build-image.yml` run from `refs/heads/main` only. A platform deployed from another revision (a working branch) must also admit the images that branch built. It therefore trusts **exactly one additional ref**, regexp-escaped, and only in that environment. Never a pattern such as `prd-.+`: another branch of the repository stays untrusted.

The flip side: after a merge, the pinned images were signed on the working branch, which `main` no longer recognises. The `release-repin.yml` workflow re-pins them in dependency order (Dockerfiles, then workloads, then fixtures, then test parameters), one pull request per step, never pushing to `main` directly.

**What not to do**: widen the committed identity to `main` or `prd-*` to avoid this re-pinning. Any branch could then sign admissible images on the environment every consumer trusts by default.

## Known findings

| Finding | Risk | Mitigation |
|---|---|---|
| Keyless signing on public Rekor rather than a KMS | Dependency on public Sigstore services; public signing metadata | Documented above; a bank signs with a KMS/HSM key or a private Sigstore |
| Write access anchors the signing identity | Whoever can push to `main` can sign admissible images | A bank adds required reviews, required checks and restricts who can push |
| OpenTofu does not see every manual change in Harbor | A replication filter edited in the UI goes unnoticed by `plan` | The Harbor UI is read-only by convention; `tofu apply -replace` restores the rule |
| `dt-bridge` reads attestations from Harbor | A tampered referrer would mislead triage (not admission, which verifies on its own) | **Fixed**: `dt-bridge` verifies the Sigstore bundles against the build identity and the calling repository |
| The local CA lacked a `keyUsage` extension | Strict X.509 clients rejected it; `dt-bridge` relaxed Python's verification | **Fixed**: the CA carries a critical `keyUsage`, strict verification stays on |
| `kubectl debug` bypassed admission | An ephemeral container escaped the workload policies | **Fixed**: the policies also cover `pods/ephemeralcontainers` |
