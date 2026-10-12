# 0005 — Configurer Harbor avec OpenTofu plutôt qu'avec harbor-cli

**En une phrase** : la configuration de Harbor (projets, réplications, robots, rétention…) est du code OpenTofu avec le provider officiel, parce que harbor-cli affichait les secrets et n'était pas idempotente.

| Statut | Date | Portée |
|---|---|---|
| Acceptée (remplace le choix initial de harbor-cli) | septembre 2026 | Projets, caches proxy, réplications, robots, immuabilité, rétention, webhooks, sécurité de déploiement |

## Contexte

Harbor lui-même est déployé par Argo CD depuis son chart Helm. Mais sa configuration vit dans sa base de données et son API. Elle doit être **déclarative**, **idempotente** (un second passage ne change rien) et ne jamais écrire un secret dans Git, dans les logs ou dans un état en clair.

Ce que l'évaluation de harbor-cli (pré-1.0) a trouvé :

- pas de `--password-stdin` qui fonctionne en pipe : le mot de passe admin finit dans les arguments du processus ou dans un prompt interactif ;
- `robot create` **affiche le secret du robot sur la sortie standard** ;
- fonctions manquantes : pas de rétention planifiée, pas d'immuabilité non interactive, pas d'aplatissement de namespace en réplication ;
- commandes impératives : l'idempotence serait à scripter autour de list/compare/create.

Les providers Crossplane pour Harbor sont abandonnés ou maintenus par une seule personne, et un control plane Crossplane coûte de la mémoire sur un nœud unique.

## Décision

- La configuration est du code OpenTofu dans `tofu/harbor`, avec le provider officiel `goharbor/harbor`, lancé par `task harbor:configure`. `task harbor:plan` ne sort en 0 que si rien ne changerait.
- Les secrets des robots sont générés dans OpenBao et passés en **écriture seule** (`secret_wo`, jamais dans l'état).
- L'état reste local, ignoré par Git, et **chiffré** avec le fournisseur de clés OpenBao Transit.
- Les filtres de réplication nomment leurs dépôts explicitement, car l'adaptateur `github-ghcr` de Harbor ne sait pas lister GHCR. Le filtre golden est dérivé du catalogue.

## Alternatives écartées

- **harbor-cli** : fuite de secrets, non idempotent, incomplet.
- **Crossplane** : providers Harbor non maintenus, coût mémoire.
- **Scripts curl sur l'API** : il faudrait réécrire ce que fait déjà le provider.

## Compromis et coût

- Le credential du cache proxy `dhi.io` et le jeton du webhook n'ont pas d'attribut en écriture seule : ce sont les secrets présents dans l'état chiffré.
- Le provider ne détecte pas toutes les modifications faites à la main (un filtre de réplication modifié dans l'UI n'apparaît pas comme une dérive ; il faut `-replace`). Voir [Retours d'expérience](../incidents.md#derive-des-filtres-de-replication-invisible-pour-opentofu).
- Faire tourner un secret de robot demande d'incrémenter `secret_wo_version`.

## Ce qui me ferait changer d'avis

Une harbor-cli 1.0 qui corrige la gestion des secrets ne suffirait pas : l'argument décisif reste le caractère déclaratif.

## Dans le code

- [`tofu/harbor/main.tf`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/tofu/harbor/main.tf)
- [`Taskfile.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/Taskfile.yml)
