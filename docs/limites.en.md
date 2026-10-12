# Known limits

What the platform does not do, or does imperfectly. I would rather say it here than have it discovered in an interview.

## What I do not claim to cover

- **High availability**: a single kind node, one instance per database. The project demonstrates a governance chain, not production infrastructure.
- **Recovery after a node stop**: after the node stops, OpenBao comes back sealed and some controllers do not reconverge on their own. Recovery is a full rebuild, about 15 minutes.
- **Trust in a regulated environment**: keyless signing depends on public Sigstore, and write access to `main` is part of the chain of trust. A bank would use a KMS/HSM key and required reviews ([ADR 0007](decisions/0007-signature-keyless.md)).
- **Bootstrap namespaces**: `kube-system`, `cilium`, `argocd` and `kyverno` are excluded from admission, by design ([threat model](modele-de-menace.md#bootstrap-namespaces-an-accepted-exclusion)).
- **Vendor signatures**: the vendor-signature rules for Cilium and Argo CD never meet a real pod, since those components run in excluded namespaces.

## Debt after the first version

| Topic | Why it matters | Direction |
|---|---|---|
| Re-pin pull requests run no CI | Opened with `GITHUB_TOKEN`, they merge digest changes without automatic checks | Open them with a GitHub App token, then make checks required |
| Unsigned tag after a failed build | The push happens before scanning and signing; a failure leaves an unsigned tag a rerun does not re-sign | Push by digest only after scanning and signing |
| `dt-bridge` delivery guarantees | Forwarded-VEX state held in memory, no retry on failure, queue capped at 200 SBOMs | Persist state, retry with backoff, let Harbor redeliver |
| `error` results in Policy Reporter | Every namespace carries one `error` result with no visible cause | Identify the faulty policy/resource pair |
| Deprecated base shown as `fail` | Reports say `fail` where admission says "warning" | Align the report with the admission outcome |
| Dashboard ergonomics | Truncated titles, runtime alerts dominated by noise, Dependency-Track projects flooded with versions | Layout pass, rule filtering, deactivating old versions |
| Grafana memory headroom | Explore on VictoriaLogs peaks at ~640 Mi for a 768 Mi limit | Re-measure after every upgrade |

## Roadmap

| Topic | Why | Direction |
|---|---|---|
| Air-gap bundle | Regulated sites often run disconnected: images, signatures, SBOMs, VEX, Sigstore root and vulnerability databases must be imported | One OCI bundle per release, imported into Harbor and verified before use |
| Gatekeeper / Rego comparison | Many banks already run OPA Gatekeeper; the choice of Kyverno should be backed up | Rewrite the workload rules in Rego (with Ratify for signatures) and compare expressiveness, test tooling and cost |
| Buildah builds | Daemonless, rootless builds, common where Docker is forbidden on agents | A `build-image.yml` variant with the same SBOM, provenance, scan and signing steps |
| Charts relocated as OCI | Helm charts come from upstream, outside Harbor governance | Relocate every chart into Harbor (signed, scanned, immutable) |
| Restart tolerance | See above | OpenBao auto-unseal, controller restarts gated on readiness |

## Upstream issues tracked

- **Dependency-Track #6132 and #6957**: matching Debian advisories (source packages) to the SBOM's binary packages. Would unlock flaw-by-flaw proof of the DHI VEX effect.
- **kyverno/sdk#125** (fix #127): `kyverno test` deadlock. Would allow merging the test suites.
- **Dependency-Track #7094**: native OpenVEX import. Would remove `dt-bridge`'s conversion module.
