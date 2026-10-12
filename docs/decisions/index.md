# Décisions

Chaque décision structurante est écrite au moment où elle est prise, avec les options écartées et ce qu'elle coûte. Le format est celui d'un ADR (*Architecture Decision Record*) : contexte, décision, alternatives, compromis, et ce qui me ferait changer d'avis.

| # | Décision | En une phrase |
|---|---|---|
| [0001](0001-format-signature-replication.md) | Bundles Sigstore et réplication des referrers | Le format de signature moderne de cosign survit à la réplication GHCR → Harbor, à condition de répliquer les tags `sha256-*`. |
| [0002](0002-kyverno-cel.md) | Kyverno en policies CEL uniquement | Les policies historiques de Kyverno disparaissent en 1.20 ; CEL est l'avenir et ressemble à l'admission native de Kubernetes. |
| [0003](0003-kubescape.md) | Kubescape plutôt que Trivy Operator + Falco | Un seul agent au lieu de deux piles, pour tenir dans 12 Gio. |
| [0004](0004-dependency-track-dt-bridge.md) | Dependency-Track avec un pont maison | Le tri des CVE se fait sur le SBOM signé au build, via un petit service qui comble ce que Dependency-Track ne sait pas faire. |
| [0005](0005-opentofu-goharbor.md) | OpenTofu plutôt que harbor-cli | La CLI Harbor affichait les secrets et n'était pas idempotente ; le provider OpenTofu l'est. |
| [0006](0006-miroir-transparent-niveaux-confiance.md) | Miroir transparent et niveaux de confiance | Harbor ne doit jamais être un point de défaillance unique, et l'admission ne doit pas bloquer ce qui la fait tourner. |
| [0007](0007-signature-keyless.md) | Signature keyless liée à `main` | Pas de clé privée à gérer ; seule la branche `main` de ce dépôt produit des images admissibles. |
| [0008](0008-divergence-grype-trivy.md) | Dependency-Track arbitre entre Grype et Trivy | Deux scanners donnent deux listes de CVE différentes ; on ne les additionne pas, on trie dans un seul outil. |
