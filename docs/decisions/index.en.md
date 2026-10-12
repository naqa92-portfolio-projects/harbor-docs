# Decisions

Every structuring decision is written down when it is made, with the options set aside and what it costs. The format is an ADR (*Architecture Decision Record*): context, decision, alternatives, trade-offs, and what would make me change my mind.

| # | Decision | In one sentence |
|---|---|---|
| [0001](0001-format-signature-replication.md) | Sigstore bundles and referrer replication | cosign's modern signature format survives GHCR → Harbor replication, as long as the `sha256-*` tags are replicated too. |
| [0002](0002-kyverno-cel.md) | Kyverno CEL policies only | Kyverno's legacy policies are removed in 1.20; CEL is the future and reads like native Kubernetes admission. |
| [0003](0003-kubescape.md) | Kubescape over Trivy Operator + Falco | One agent instead of two stacks, to fit in 12 GiB. |
| [0004](0004-dependency-track-dt-bridge.md) | Dependency-Track with an in-house bridge | CVE triage runs on the SBOM signed at build time, through a small service filling what Dependency-Track cannot do. |
| [0005](0005-opentofu-goharbor.md) | OpenTofu over harbor-cli | The Harbor CLI printed secrets and was not idempotent; the OpenTofu provider is. |
| [0006](0006-miroir-transparent-niveaux-confiance.md) | Transparent mirror and trust tiers | Harbor must never be a single point of failure, and admission must not block what runs it. |
| [0007](0007-signature-keyless.md) | Keyless signing bound to `main` | No private key to manage; only this repository's `main` branch produces admissible images. |
| [0008](0008-divergence-grype-trivy.md) | Dependency-Track arbitrates between Grype and Trivy | Two scanners give two different CVE lists; we never add them up, we triage in one tool. |
