# Next steps

The scope of the first version, and what I would do next. Each item says why it matters and which direction to take.

## Scope of the first version

- **Single-node demonstrator**: one kind cluster, one instance per database. The subject is the governance chain; high availability belongs to the infrastructure that would host it.
- **Recovery by rebuild**: after a node stop, the documented recovery is `task down && task up` (about 15 minutes).
- **Public keyless signing**: suited to a public repository; in a regulated environment, a KMS/HSM key or a private Sigstore takes over without changing the policies beyond the attestor ([ADR 0007](decisions/0007-signature-keyless.md)).
- **Bootstrap namespaces excluded from admission**, by design, to avoid any dependency loop ([threat model](modele-de-menace.md#bootstrap-namespaces-an-accepted-exclusion)).

## Identified improvements

| Topic | Why | Direction |
|---|---|---|
| CI on re-pin pull requests | Pull requests opened by workflows with `GITHUB_TOKEN` do not trigger CI | Open them with a GitHub App token, then make checks required |
| Publishing by digest | Publishing the image only after scanning and signing simplifies reruns | Push by digest once scanning and signing pass |
| `dt-bridge` delivery | Today forwarded-VEX state lives in memory and the queue is capped at 200 SBOMs | Persist state, retry with backoff, rely on Harbor redelivery |
| OpenBao auto-unseal | Would allow recovery after a node stop without a rebuild | Transit or static unseal, controller restarts gated on readiness |
| Policy Reporter results | One `error` result shows per namespace for the image policies; the deprecated base shows as `fail` | Identify the policy/resource pair involved; align the report with the admission outcome |
| Dashboard | Truncated titles, runtime alerts to filter, old versions to archive in Dependency-Track | Layout and filtering pass |
| Vendor signatures at admission | Cilium and Argo CD run in excluded namespaces | Verify their images at bootstrap, or move them out of the excluded tier once Kyverno is up |

## Roadmap

| Topic | Why | Direction |
|---|---|---|
| Air-gap bundle | Regulated sites often run disconnected: images, signatures, SBOMs, VEX, Sigstore root and vulnerability databases must be imported | One OCI bundle per release, imported into Harbor and verified before use |
| Gatekeeper / Rego comparison | Many banks already use OPA Gatekeeper | Rewrite the workload rules in Rego (with Ratify for signatures) and compare expressiveness, test tooling and cost |
| Buildah builds | Daemonless, rootless builds, common where Docker is forbidden on agents | A `build-image.yml` variant with the same SBOM, provenance, scan and signing steps |
| Charts relocated as OCI | Extend Harbor governance to Helm charts | Relocate every chart into Harbor (signed, scanned, immutable) |

## Upstream changes tracked

- **Dependency-Track #6132 and #6957**: matching Debian advisories (source packages) to the SBOM's binary packages. Will make the effect of DHI VEX visible flaw by flaw.
- **kyverno/sdk#125** (fix #127): will allow merging the `kyverno test` suites.
- **Dependency-Track #7094**: native OpenVEX import; `dt-bridge`'s conversion module can then be removed.
