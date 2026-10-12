# What broke

What I ran into while building the platform: upstream bugs worked around, races between components, and my own mistakes. Each entry follows the same thread: **symptom → cause → fix → lesson**. Every workaround still in place has an exit condition: the day upstream fixes it, we know what to remove.

## What taught me the most

### The Harbor provider does not see filter drift

- **Symptom**: a replication filter edited in the Harbor UI; `tofu plan` stays empty, `apply` repairs nothing.
- **Cause**: the `goharbor/harbor` provider does not read every filter attribute back into its state (goharbor/terraform-provider-harbor#422, closed without a fix). With a single `filters` block, it even loses the attribute that is not `name`.
- **Fix**: two separate `filters` blocks (one `name`, one `tag`), and `tofu apply -replace` for any change made outside the code.
- **Lesson**: "declarative" does not mean "detects all drift". You need to know what the provider actually reads back, and treat the UI as read-only.

### The proxy cache returns a different digest than upstream

- **Symptom**: an image pulled through the Harbor cache shows up minutes later, under a different digest than on Docker Hub.
- **Cause**: for a multi-arch index pulled for a single platform, Harbor waits for the other platforms, then pushes a **trimmed index** with a new digest.
- **Fix**: cache checks verify the platform manifest digest, not the tag, and tolerate minutes of delay.
- **Lesson**: reading the upstream source (here `manifestcache.go`) answers faster than stacking hypotheses.

### `kubectl debug` bypassed admission

- **Symptom**: an ephemeral container added with `kubectl debug` escaped the workload policies.
- **Cause**: the policies only targeted `pods`, not the `pods/ephemeralcontainers` subresource.
- **Fix**: every workload policy and `tier-required` also cover ephemeral containers. Verified with a dry run on the cluster.
- **Lesson**: an admission policy must be tested against every way of running code, not only pod creation.

### The cluster does not recover from a stopped node

- **Symptom**: after `docker stop` then `start` of the kind node, the Kyverno reports controller crash-loops, and Argo CD keeps some Applications degraded while their children are healthy.
- **Cause**: OpenBao comes back sealed, and the reports controller, under its 128 Mi memory limit, loses its leader lease.
- **Fix**: memory limit raised; the documented recovery is a full rebuild (`task down && task up`). The README was corrected: it wrongly claimed `task up` would converge a stopped node.
- **Lesson**: writing down what the platform **cannot** do matters as much as the rest. Restart tolerance is on the [roadmap](limites.md).

### Kyverno stuck for 12 minutes at startup

- **Symptom**: on a fresh cluster, the Kyverno pods stay stuck for about 12 minutes on a ConfigMap mount.
- **Cause**: the local CA ConfigMap is produced by trust-manager, created later than Kyverno.
- **Fix**: Argo CD wave order revised, health checks on the Kyverno policies, Pod Security labels set declaratively. Kyverno still waits about 2 minutes 30.
- **Lesson**: in a GitOps bootstrap, creation order matters as much as content. Hence the invariant "no workload before admission".

## Upstream bugs worked around

| Upstream | Symptom | Workaround | Exit condition |
|---|---|---|---|
| `kyverno test` (kyverno/sdk#125) | The CLI deadlocks when two image-verifying policies evaluate the same resource | One test suite per policy | Fix kyverno/sdk#127 shipped in a CLI release |
| Dependency-Track (#6132, #6957) | No Debian flaw matched to the SBOM's binary packages | VEX conversion targets the binary package via the `upstream=` qualifier | Dependency-Track handles the source → binary link |
| Dependency-Track (#7094) | No OpenVEX import | OpenVEX → CycloneDX conversion isolated in one module | Native OpenVEX import |
| CycloneDX (specification#609) | No equivalent for the "cannot be controlled by an adversary" justification | The reason is kept in the free-text field | The specification adds the value |
| Harbor's `github-ghcr` adapter | An "everything under this owner" filter replicates nothing: the adapter cannot list GHCR | Every repository named explicitly, golden filter derived from the catalog | The adapter can list a GHCR owner |
| Harbor chart | The `quay` proxy cache type is missing from the chart's list | Environment variable overriding the list | The chart includes `quay` |
| Kubescape storage | Listing `ContainerProfile` objects returns metadata without the `spec` | Argo CD ignores `/spec`; a hash annotation acts as the sync trigger | Kubescape documents or removes this requirement |
| Kubescape node-agent | The first shell can arrive before the profile is loaded: no alert | The demo waits for the profile-loaded log line and retries a shell if needed; a hand-written profile is adopted immediately | node-agent exposes a per-pod readiness signal |
| Cilium Gateway API | A port restriction on an egress rule towards a Gateway-routed backend causes 403s | Egress rule without a port restriction for that single hop | Cilium keeps the destination port when checking |
| Argo CD on WSL2 | "counter cannot decrease" panic when the host clock steps back | `ServerSideApply=true` on most Applications; occasional manual resync | Upstream fix, or a host whose clock does not step back |
| GitHub Actions | Pull requests opened with `GITHUB_TOKEN` trigger no CI | No required check; the maintainer reruns CI by hand | Pull requests opened with a GitHub App token |
| Policy Reporter | The UI API does not return the source of each result | Tests filter with `sources=` | The API exposes the field |

## Deliberate configuration choices

- **`tomcat.version` forced in `hello-java`** above the version managed by Spring Boot, to close a CVE. Renovate tracks it with a dedicated rule. To be removed when Spring Boot ships the fixed version.
- **`config.preserve: false` on Kyverno**: the chart's post-delete hook prevents Argo CD from pruning the Application. Kyverno is always installed and removed by Argo CD here; nothing depends on that configuration after uninstall.
- **Memory limits tightened, then loosened**: VictoriaLogs was OOM-killed at 160 Mi, a limit set when the budget targeted 10 GiB and kept after the move to 12 GiB. Grafana was raised to 768 Mi after an OOM in Explore. Lesson: a limit set under a constraint must be re-measured when the constraint changes.
