# Lessons learned

The problems met while building the platform, and how they were handled. Each entry follows the same thread: **detected → understood → fixed → safeguard**. When the fix depends on an upstream project, the condition for removing the workaround is written down: the day upstream fixes it, we know what to remove.

## Notable cases

### Replication filter drift invisible to OpenTofu

- **Detected**: a replication filter edited in the Harbor UI did not show up in `tofu plan`.
- **Understood**: the `goharbor/harbor` provider does not read every filter attribute back into its state (goharbor/terraform-provider-harbor#422). With a single `filters` block, the attribute other than `name` is even lost.
- **Fixed**: two separate `filters` blocks (one `name`, one `tag`); `tofu apply -replace` for any change made outside the code.
- **Safeguard**: the Harbor UI is read-only by convention, and the workaround is documented with its exit condition.
- **What I take from it**: "declarative" does not guarantee that all drift is detected; you need to know what the provider actually reads back.

### Different digest between the proxy cache and upstream

- **Detected**: an image pulled through the Harbor cache appeared minutes later, under a different digest than on Docker Hub.
- **Understood**: by reading Harbor's code (`manifestcache.go`): for a multi-arch index pulled for a single platform, Harbor waits for the other platforms, then pushes a trimmed index with a new digest.
- **Fixed**: cache checks verify the platform manifest digest, not the tag, and tolerate this delay.
- **What I take from it**: reading upstream code settles things faster than stacking hypotheses.

### Admission coverage of ephemeral containers

- **Detected**: in a security review, before delivery, an ephemeral container added with `kubectl debug` was not evaluated by the workload policies.
- **Understood**: the policies targeted `pods`, not the `pods/ephemeralcontainers` subresource.
- **Fixed**: every workload policy and `tier-required` also cover ephemeral containers, verified with a dry run on the cluster.
- **Safeguard**: a Chainsaw regression scenario checks that a non-compliant ephemeral container is denied.
- **What I take from it**: an admission policy must be tested against every way of running code, not only pod creation.

### Recovery after a node stop

- **Detected**: in review, by stopping then restarting the kind node: the Kyverno reports controller did not resume, and OpenBao came back sealed.
- **Understood**: under its 128 Mi memory limit, the controller lost its leader lease; OpenBao reseals on every restart by design.
- **Fixed**: memory limit raised; the documented recovery procedure is a rebuild (`task down && task up`, about 15 minutes), and the README was aligned with this behaviour.
- **Next**: OpenBao auto-unseal is in the [next steps](limites.md).

### Startup order between trust-manager and Kyverno

- **Detected**: on a fresh cluster, the Kyverno pods waited about 12 minutes for a ConfigMap mount.
- **Understood**: the local CA ConfigMap is produced by trust-manager, created after Kyverno.
- **Fixed**: Argo CD wave order revised, health checks on the Kyverno policies, Pod Security labels set declaratively. The wait dropped to about 2 minutes 30.
- **Safeguard**: the invariant "no workload before admission" is carried by the wave order.

## Workarounds for upstream limits

| Upstream | Finding | Workaround in place | Removal condition |
|---|---|---|---|
| `kyverno test` (kyverno/sdk#125) | The CLI deadlocks when two image-verifying policies evaluate the same resource | One test suite per policy | Fix kyverno/sdk#127 shipped in a CLI release |
| Dependency-Track (#6132, #6957) | Debian advisories (source packages) are not matched to the SBOM's binary packages | VEX conversion targets the binary package via the `upstream=` qualifier | Dependency-Track handles the source → binary link |
| Dependency-Track (#7094) | No OpenVEX import | OpenVEX → CycloneDX conversion isolated in one module | Native OpenVEX import |
| CycloneDX (specification#609) | No equivalent for the "cannot be controlled by an adversary" justification | The reason is kept in the free-text field | The specification adds the value |
| Harbor's `github-ghcr` adapter | The adapter cannot list a GHCR owner | Every repository is named explicitly, golden filter derived from the catalog | The adapter can list an owner |
| Harbor chart | The `quay` proxy cache type is missing from the chart's list | Environment variable completing the list | The chart includes `quay` |
| Kubescape storage | Listing `ContainerProfile` objects does not return the `spec` | Argo CD ignores `/spec`; a hash annotation triggers the sync | Kubescape documents or removes this requirement |
| Kubescape node-agent | A new container's profile can be loaded after the container starts | The demo waits for the profile to load; a hand-written profile is adopted immediately | node-agent exposes a per-pod readiness signal |
| Cilium Gateway API | A port restriction on an egress rule towards a Gateway-routed backend causes 403s | Egress rule without a port restriction for that single hop | Cilium keeps the destination port when checking |
| Argo CD on WSL2 | "counter cannot decrease" panic when the host clock steps back | `ServerSideApply=true` on most Applications | Upstream fix |
| GitHub Actions | Pull requests opened with `GITHUB_TOKEN` do not trigger CI | The maintainer reruns CI on the branch | Pull requests opened with a GitHub App token |
| Policy Reporter | The UI API does not return the source of each result | Tests filter with `sources=` | The API exposes the field |

## Documented configuration choices

- **`tomcat.version` forced in `hello-java`** above the version managed by Spring Boot, to close a CVE. Renovate tracks it with a dedicated rule; to be removed when Spring Boot ships the fixed version.
- **`config.preserve: false` on Kyverno**: the chart's post-delete hook prevents Argo CD from pruning the Application; here Kyverno is always installed and removed by Argo CD.
- **Memory limits re-measured**: VictoriaLogs (160 Mi) and Grafana (768 Mi today) were readjusted when the overall budget moved from 10 to 12 GiB. A limit set under a constraint gets re-measured when the constraint changes.
