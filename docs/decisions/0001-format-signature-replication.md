# 0001 — Signer au format bundle Sigstore et répliquer les referrers dans Harbor

**En une phrase** : le format de signature moderne de cosign (bundles Sigstore) survit à la réplication GHCR → Harbor et est lu par Kyverno, à condition de répliquer aussi les tags `sha256-*` qui portent les signatures.

| Statut | Date | Portée |
|---|---|---|
| Acceptée (point 4 remplacé, voir plus bas) | septembre 2026, spike M0 | Toute image signée dans GitHub Actions, répliquée dans Harbor et admise par Kyverno |

## Contexte

Les images sont construites dans GitHub Actions, signées et attestées avec cosign, poussées sur GHCR, répliquées dans Harbor, puis admises par Kyverno sur leur référence Harbor. Il existe deux façons de stocker une signature :

- l'ancien format cosign, des tags `sha256-<digest>.sig` et `.att` ;
- les **bundles Sigstore** attachés comme *referrers* OCI (des artefacts qui pointent vers l'image qu'ils décrivent).

Avant de construire quoi que ce soit, un spike devait prouver lequel survit à la réplication et lequel Kyverno sait lire.

## Décision

1. **Format bundle Sigstore** (le défaut de cosign v3). Les tags `.sig`/`.att` sont dépréciés en cosign v3 ; la plateforme ne construit pas dessus.
2. **La provenance SLSA v1 est écrite par le workflow lui-même** et attachée avec `cosign attest`. Elle enregistre le digest de l'image de base, que la policy de catalogue lit ensuite. L'action `actions/attest-build-provenance` n'est pas utilisée : elle stocke dans l'API d'attestations de GitHub et signe avec une autre identité que le workflow.
3. **La réplication Harbor copie le tag de repli des referrers.** Les règles des projets `golden` et `apps` n'ont pas de filtre de tag, ce qui réplique les tags `sha256-*` avec les images.
4. ~~Kyverno fait confiance à la CA de Harbor via `SSL_CERT_DIR` et une ConfigMap montée à la main.~~ **Remplacé** : c'est désormais **trust-manager** qui produit la ConfigMap `kyverno-harborlab-ca` à partir de la CA locale ; Kyverno la monte à côté des racines système, toujours via `SSL_CERT_DIR`.

## Ce que le spike a mesuré

| Sujet | Observé |
|---|---|
| Stockage GHCR | GHCR répond 404 sur l'API referrers. cosign écrit donc un tag `sha256-<hex>` contenant un index des bundles (une signature + une entrée par attestation). |
| Réplication | Harbor copie l'image avec un digest identique. Avec le filtre `{m0,sha256-*}` il copie aussi l'index des signatures et le sert sur sa propre API referrers. **Un filtre sur le seul tag de l'image fait perdre toutes les signatures.** |
| Kyverno | `ImageValidatingPolicy` détecte le format bundle et lit la signature et les trois attestations via Harbor. L'image non signée est refusée avec un message qui nomme la policy. |
| Confiance CA de Kyverno | Le champ `caCertificates` du chart **remplace** le magasin de certificats. Y ajouter tout le bundle système dépasse la limite de 1 Mio du Secret de release Helm, et l'omettre casse le TLS vers Sigstore. D'où le montage à côté des racines système. |
| Remplissage du cache proxy | Asynchrone. Pour un index multi-architecture tiré pour une seule plateforme, Harbor attend jusqu'à 20 × 20 s les autres plateformes, puis pousse **un index réduit avec un nouveau digest**. Mesuré : environ 5 minutes avant que le tag apparaisse. |

## Alternatives écartées

- **Tags `.sig`/`.att`** : dépréciés, voués à disparaître.
- **`actions/attest-build-provenance`** : identité de signature différente du workflow, stockage hors registre.
- **`caCertificates` du chart Kyverno** : remplace les racines système, casse Sigstore.

## Compromis et coût

- Les règles de réplication ne peuvent pas filtrer finement les tags sans perdre les signatures.
- Le cache proxy peut produire des digests différents de l'amont pour les index multi-architecture : un contrôle par tag doit tolérer des minutes de délai.
- Les clients du registre et Kyverno ont besoin de la CA de la plateforme, générée à l'exécution et jamais commitée.

## Ce qui me ferait changer d'avis

GHCR qui implémente l'API referrers OCI : le tag de repli deviendrait inutile.

## Dans le code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`tofu/harbor/main.tf`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/tofu/harbor/main.tf)
- [`platform/apps/templates/kyverno.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/kyverno.yaml)
- [`platform/apps/templates/trust-manager.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/trust-manager.yaml)
- [`spike`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/spike)
