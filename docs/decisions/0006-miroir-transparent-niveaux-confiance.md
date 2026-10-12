# 0006 — Miroir de registre transparent et niveaux de confiance par namespace

**En une phrase** : tous les pulls passent par Harbor, mais avec repli automatique vers l'amont, pour que Harbor ne soit jamais un point de défaillance unique ; et l'admission n'est pas imposée aux composants qui la font tourner.

| Statut | Date | Portée |
|---|---|---|
| Acceptée | septembre 2026 | Comment les nœuds tirent les images, et ce que l'admission exige selon le namespace |

## Contexte

On veut que chaque pull passe par Harbor (cache, scan, un seul point de sortie). Mais Harbor, sa base de données, le CNI, le contrôleur GitOps et le moteur de policies tournent sur le même cluster. S'ils ne pouvaient tirer leurs images que via Harbor, **une panne de Harbor empêcherait Harbor de redémarrer**.

De même, imposer la chaîne de confiance complète aux composants qui implémentent l'admission créerait une boucle de dépendance.

## Décision

- **Miroir transparent** : containerd, sur le nœud, a des miroirs `hosts.toml` pour `docker.io`, `quay.io`, `ghcr.io`, `registry.k8s.io` et `dhi.io` qui pointent vers le projet cache de Harbor correspondant, **avec l'amont en repli**. Les manifestes gardent leurs références d'origine ; si Harbor est en panne, le pull repart vers l'amont. Le bootstrap n'a besoin d'aucun cas particulier.
- **Niveaux de confiance** par label de namespace :
    - `workload` : chaîne complète, bloquante ;
    - `platform` : liste de registres autorisés et signatures éditeur, en mode `Audit` (signalé, pas bloqué) ;
    - sans niveau : tout pod refusé, pour qu'un label oublié ne désactive jamais les contrôles ;
    - bootstrap (`kube-system`, `cilium`, `argocd`, `kyverno`) : exclus nommément, voir le [modèle de menace](../modele-de-menace.md#namespaces-de-bootstrap-une-exclusion-assumee).

## Alternatives écartées

- **Réécrire toutes les références vers Harbor** (`harbor…/dockerhub-proxy/…`) : Harbor devient un point de défaillance unique pour son propre redémarrage.
- **Même niveau d'exigence partout** : boucle de dépendance, un cluster qui ne peut plus converger après une panne.

## Compromis et coût

- Les workloads doivent référencer Harbor `golden`/`apps` explicitement : une référence `docker.io` servie par le miroir reste refusée, car l'admission voit la référence, pas le chemin réel du pull.
- Les composants plateforme gardent leurs références amont et leurs signatures éditeur ; leurs écarts sont visibles sans bloquer la reprise.
- L'exclusion du bootstrap est un risque accepté, limité aux administrateurs du cluster.
- Le remplissage du cache est asynchrone et peut réduire les index multi-architecture ([ADR 0001](0001-format-signature-replication.md)).

## Ce qui me ferait changer d'avis

Un cluster de production avec un registre hautement disponible hors cluster : on pourrait alors exiger le passage par le registre pour tout le monde.

## Dans le code

- [`scripts/platform-up.sh`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/scripts/platform-up.sh)
- [`platform/kind/cluster.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/kind/cluster.yaml)
- [`policies/tiers`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/policies/tiers)
- [`platform/config/tiers/namespaces.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/platform/config/tiers/namespaces.yaml)
