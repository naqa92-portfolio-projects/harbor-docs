# 0004 — Trier les vulnérabilités dans Dependency-Track, alimenté par un pont maison

**En une phrase** : Dependency-Track est la source de vérité du tri des CVE, alimenté avec le SBOM signé au build par `dt-bridge`, un petit service Python qui comble ce que Dependency-Track ne sait pas faire seul.

| Statut | Date | Portée |
|---|---|---|
| Acceptée | septembre 2026 | Stockage des SBOM, tri des vulnérabilités, gestion des VEX |

## Contexte

Chaque image porte un SBOM CycloneDX attesté au build, et sa base golden vient avec des déclarations OpenVEX de l'éditeur (Docker Hardened Images publie des déclarations `not_affected`). Kubescape produit des VEX runtime.

Le tri a besoin d'un seul endroit où voir les failles par version d'image et appliquer les VEX, **alimenté par le SBOM attesté, pas par un SBOM régénéré**. Or Dependency-Track 5.1 importe les SBOM et VEX CycloneDX, mais **pas OpenVEX** (DependencyTrack/dependency-track#7094), et n'a aucune intégration Harbor.

## Décision

- Dependency-Track 5.1, sur CloudNativePG, est la source de vérité du tri.
- `dt-bridge`, un service FastAPI construit lui-même par le golden path, fait le lien :
    - à chaque webhook Harbor, il lit l'attestation CycloneDX de l'image, **vérifie sa signature contre l'identité de build**, et envoie le SBOM inchangé au projet `<projet harbor>/<dépôt>`, version = tag. Si la provenance ne se vérifie pas, rien n'est envoyé ;
    - il lit la base DHI dans la provenance vérifiée, récupère son OpenVEX, convertit les déclarations `not_affected` en VEX CycloneDX et les envoie une fois l'analyse Dependency-Track terminée ;
    - il transmet de la même façon les VEX runtime de Kubescape ;
    - il expose les métriques de posture du tableau de bord Grafana.
- Tout le vocabulaire OpenVEX est isolé dans un module (`vex.py`), supprimable d'un bloc le jour où Dependency-Track importera OpenVEX.

## Alternatives écartées

- **Laisser Harbor seul trier** : Harbor scanne, mais ne gère ni le cycle de vie du tri ni les VEX par composant.
- **Régénérer un SBOM côté Dependency-Track** : on trierait autre chose que ce qui a été signé au build.

## Compromis et coût

- Les SBOM dans Dependency-Track sont exactement ceux attestés et signés au build : c'est le bénéfice principal.
- **Limite amont** : Dependency-Track rapproche les avis Debian par nom de paquet *source*, alors que le SBOM liste les paquets *binaires* (DependencyTrack/dependency-track#6132, #6957). Aucune faille Debian n'apparaît donc pour les images golden, et l'effet d'un VEX DHI au niveau de chaque faille n'est pas encore observable. L'envoi du VEX, lui, est accepté et traité (`COMPLETED`).
- `dt-bridge` est un composant de plus à maintenir, avec ses propres tests (26 tests pytest).

## Ce qui me ferait changer d'avis

Dependency-Track qui importe OpenVEX nativement : `vex.py` disparaît.

## Dans le code

- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
- [`platform/workloads/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/platform/workloads/dt-bridge)
- [`platform/apps/templates/dependency-track.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/dependency-track.yaml)
