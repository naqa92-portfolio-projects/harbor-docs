# 0008 — Grype et Trivy divergent : Dependency-Track fait foi

**En une phrase** : trois scanners voient les mêmes images et donnent des listes de CVE différentes ; on ne les réconcilie pas et on ne les additionne jamais, on trie dans Dependency-Track.

| Statut | Date | Portée |
|---|---|---|
| Acceptée | septembre 2026 | Quels comptes de vulnérabilités font autorité |

## Contexte

Trois scanners voient les mêmes images :

- **Trivy en CI** (`build-image.yml`), bloquant, avec les VEX de l'image de base ;
- **Trivy dans Harbor** (scan au push, sécurité de déploiement) ;
- **Grype dans le cluster** (Kubescape), sur les images qui tournent.

Trivy et Grype n'ont ni les mêmes bases de vulnérabilités, ni les mêmes algorithmes de correspondance, ni la même façon d'appliquer les VEX. Leurs listes divergent, parfois sur la sévérité, parfois sur la présence même d'une CVE. Dependency-Track rapproche le SBOM attesté de ses propres sources.

## Décision

- Trivy reste la barrière de build (CI) et de registre (Harbor).
- Grype reste la vue propre de Kubescape sur les images qui tournent ; elle n'est pas réconciliée avec Trivy.
- Dependency-Track, alimenté par `dt-bridge`, est la source de vérité du tri. Le tableau de bord *image posture* étiquette chaque compte de CVE avec sa source (`harbor-trivy`, `dependency-track`), avant et après VEX.

## Alternatives écartées

- **Un seul scanner partout** : Kubescape embarque Grype, et Trivy est le scanner natif de Harbor ; remplacer l'un ou l'autre ferait perdre une intégration.
- **Additionner ou fusionner les comptes** : on compterait deux fois la même faille, ou on masquerait un désaccord utile.

## Compromis et coût

- Une CVE peut bloquer un build dans Trivy et être absente de Grype, ou l'inverse. Chaque outil se lit selon ses propres règles, et le tri se fait dans Dependency-Track.
- Le tableau de bord n'additionne jamais les comptes de deux scanners.

## Dans le code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`platform/observability/grafana/image-posture.json`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/observability/grafana/image-posture.json)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
