# 0003 — Kubescape plutôt que Trivy Operator et Falco dans le cluster

**En une phrase** : un seul opérateur (Kubescape) couvre le scan des images en cours d'exécution, les contrôles de posture et la détection runtime, là où la combinaison habituelle en demande deux, ce qui tient dans un budget de 12 Gio.

| Statut | Date | Portée |
|---|---|---|
| Acceptée | septembre 2026 | Scan des vulnérabilités dans le cluster, posture, détection runtime, VEX runtime |

## Contexte

Il faut, à l'intérieur du cluster :

- scanner les images qui tournent ;
- vérifier la configuration (contrôles CIS) ;
- détecter un comportement anormal à l'exécution (**un shell ouvert dans un conteneur doit lever une alerte**) ;
- idéalement, prouver qu'un paquet vulnérable n'est jamais chargé.

Le duo classique est Trivy Operator (scan, posture) plus Falco (règles runtime) : deux opérateurs, deux agents, deux langages de règles, sur un nœud unique limité à 12 Gio.

## Décision

L'opérateur Kubescape remplace les deux :

- scan des images en cours d'exécution avec Grype ;
- contrôles CIS et NSA ;
- détection runtime par l'agent eBPF `node-agent`, qui compare le comportement de chaque conteneur à un profil (appris, ou écrit à la main comme pour `runtime-demo`) ; les alertes sont des lignes de log JSON collectées par VictoriaLogs ;
- documents OpenVEX runtime (expérimental) qui listent les paquets jamais chargés, transmis par `dt-bridge` à Dependency-Track.

## Alternatives écartées

- **Trivy Operator + Falco** : deux piles complètes pour la même couverture, trop lourd pour le budget mémoire.
- **Falco seul** : excellent en runtime, mais ne couvre ni le scan ni la posture.

## Compromis et coût

- Le scanner du cluster (Grype) n'est pas celui de la CI et de Harbor (Trivy) : leurs listes de CVE divergent. Dependency-Track tranche ([ADR 0008](0008-divergence-grype-trivy.md)).
- La génération de VEX runtime est expérimentale : environ deux minutes d'observation par conteneur, et des noms d'images incohérents (`docker.io` contre `index.docker.io`) que `dt-bridge` normalise.
- Une règle runtime reste muette tant que le profil du conteneur n'est pas chargé. Un profil appris n'existe qu'après la période d'apprentissage ; un profil écrit à la main existe tout de suite. D'où le profil écrit à la main pour la démo.
- On perd le grand catalogue de règles communautaires de Falco. Les règles par défaut de Kubescape couvrent les cas de démo.

## Ce qui me ferait changer d'avis

Un besoin de règles runtime fines et nombreuses (conformité PCI par exemple) : Falco redeviendrait pertinent, quitte à payer la mémoire.

## Dans le code

- [`platform/apps/templates/kubescape.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/apps/templates/kubescape.yaml)
- [`platform/workloads/runtime-demo`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/platform/workloads/runtime-demo)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
