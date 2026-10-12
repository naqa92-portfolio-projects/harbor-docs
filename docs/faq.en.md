# Interview questions

The questions I get asked, or should get asked, with the short answer I give out loud. Each answer links to the page with the details.

## The project

??? question "In one sentence, what is it?"
    A platform guaranteeing that only images built, signed and attested by the platform itself can run on the cluster, and tracking their vulnerabilities all the way to runtime. An application team only writes `FROM <golden image>` and one workflow line.

??? question "Why a single kind cluster and not a real cloud?"
    So anyone can run it again with 16 GiB of RAM, without a cloud account or a bill. The subject is the governance chain, not the infrastructure. Everything shown here (signatures, admission, triage) carries over as is to a managed cluster.

??? question "A non-obvious security point?"
    Binding the signing identity to the repository. The build workflow is reusable and public: without a check, any repository calling it would get the same signing identity. Admission therefore requires the signed provenance to name this repository as the caller. This flaw was found while reviewing the threat model, then fixed. → [ADR 0007](decisions/0007-signature-keyless.md)

## Technical choices

??? question "Why Kyverno rather than OPA Gatekeeper?"
    Kyverno verifies image signatures natively, and its CEL policies look like native Kubernetes admission. Gatekeeper needs an external provider (Ratify) for signatures and one more language, Rego. Since many banks already run Gatekeeper, a side-by-side comparison is on the roadmap. → [ADR 0002](decisions/0002-kyverno-cel.md)

??? question "Why sign without a key? A bank would not do that."
    True, and it is written down. In a public repository without a KMS, a long-lived key would be one more secret to protect, for nothing. Keyless binds the signature to the workflow identity and makes it auditable in Rekor. In a bank, I would switch to a KMS or HSM key: only the attestor definition changes in the policies. → [ADR 0007](decisions/0007-signature-keyless.md)

??? question "Why read the base from the provenance rather than from an image label?"
    A label is declarative: anyone can write anything in a Dockerfile. The SLSA provenance is produced and signed by the build workflow, which actually saw the base used. → [Architecture](architecture.md)

??? question "Why Kubescape rather than Falco?"
    Memory budget. Falco only covers runtime; Trivy Operator would have been needed on top for scanning and posture. Kubescape does all three with a single agent. I lose Falco's large rule catalogue, which is acceptable here. → [ADR 0003](decisions/0003-kubescape.md)

??? question "Why write an in-house service, dt-bridge?"
    Dependency-Track has no Harbor integration and does not import OpenVEX. I wanted it to triage the SBOM signed at build time, not a regenerated one. dt-bridge verifies the signature, uploads the SBOM and converts the VEX. The conversion code is isolated so it can be removed in one piece. → [ADR 0004](decisions/0004-dependency-track-dt-bridge.md)

??? question "Why OpenTofu to configure Harbor?"
    harbor-cli printed robot secrets on standard output and was not idempotent. The official provider is declarative, secrets go write-only and the state is encrypted. Its limit: it does not see every change made in the UI. → [ADR 0005](decisions/0005-opentofu-goharbor.md)

??? question "What happens if Harbor goes down?"
    Pulls fall back upstream thanks to the transparent containerd mirror. Harbor is never needed for its own restart. Workloads, on the other hand, reference Harbor explicitly: they wait for it to come back. → [ADR 0006](decisions/0006-miroir-transparent-niveaux-confiance.md)

??? question "Why exclude kube-system, cilium, argocd and kyverno from admission?"
    They are the components admission depends on. If a policy mistake or a Rekor outage blocked their pods, the cluster could no longer repair itself. It is an accepted, written-down risk, limited to administrators. → [Threat model](modele-de-menace.md#bootstrap-namespaces-an-accepted-exclusion)

??? question "Trivy and Grype report different CVEs: which one do you trust?"
    Neither alone. Trivy is the gate at build time and in Harbor, Grype is the cluster's view, and triage happens in Dependency-Track. The dashboard labels each count with its source and never adds up two scanners. → [ADR 0008](decisions/0008-divergence-grype-trivy.md)

## What broke, what I learned

??? question "Which bug taught you the most?"
    Harbor's proxy cache returning a different digest than Docker Hub. The explanation came from Harbor's own code: for a multi-arch index, it waits for the other architectures, then pushes a trimmed index with a new digest. Lesson: read upstream code rather than stack hypotheses. → [What broke](incidents.md)

??? question "A flaw found in review?"
    `kubectl debug` could add an ephemeral container that escaped admission: the policies did not cover the `pods/ephemeralcontainers` subresource. Fixed and tested. A policy must be tested against every way of running code. → [What broke](incidents.md#kubectl-debug-bypassed-admission)

??? question "What would you change if you started over?"
    I would handle restart recovery from the start (OpenBao auto-unseal) instead of documenting a full rebuild. And I would open the automated pull requests with a GitHub App token from day one, so CI runs on them. → [Known limits](limites.md)

??? question "How do you know it works?"
    143 bats tests, 17 Chainsaw scenarios on an ephemeral cluster, 8 `kyverno test` suites and 26 pytest tests, plus nine demo scenarios that only succeed if they observe the expected reaction. → [Demo](demo.md)
