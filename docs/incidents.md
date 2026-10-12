# Ce qui a cassé

Ce que j'ai rencontré en construisant la plateforme : des bugs amont contournés, des courses entre composants, et mes propres erreurs. Chaque entrée suit le même fil : **symptôme → cause → correctif → leçon**. Chaque contournement encore en place a une condition de sortie : le jour où l'amont corrige, on sait quoi retirer.

## Ce qui m'a le plus appris

### Le provider Harbor ne voit pas la dérive des filtres

- **Symptôme** : un filtre de réplication modifié dans l'UI Harbor ; `tofu plan` reste vide, `apply` ne répare rien.
- **Cause** : le provider `goharbor/harbor` ne relit pas tous les attributs de filtre dans son état (goharbor/terraform-provider-harbor#422, fermé sans correctif). Avec un seul bloc `filters`, il perd même l'attribut qui n'est pas `name`.
- **Correctif** : deux blocs `filters` séparés (un `name`, un `tag`), et `tofu apply -replace` pour toute modification faite hors du code.
- **Leçon** : « déclaratif » ne veut pas dire « détecte toute dérive ». Il faut savoir ce que le provider relit vraiment, et traiter l'UI comme en lecture seule.

### Le cache proxy renvoie un autre digest que l'amont

- **Symptôme** : une image tirée via le cache Harbor apparaît plusieurs minutes plus tard, sous un digest différent de Docker Hub.
- **Cause** : pour un index multi-architecture tiré pour une seule plateforme, Harbor attend les autres plateformes, puis pousse un **index réduit** avec un nouveau digest.
- **Correctif** : les contrôles du cache vérifient le digest du manifeste de la plateforme, pas le tag, et tolèrent des minutes de délai.
- **Leçon** : lire le code source de l'amont (ici `manifestcache.go`) répond plus vite qu'empiler des hypothèses.

### `kubectl debug` contournait l'admission

- **Symptôme** : un conteneur éphémère ajouté par `kubectl debug` échappait aux policies workload.
- **Cause** : les policies ne ciblaient que `pods`, pas la sous-ressource `pods/ephemeralcontainers`.
- **Correctif** : toutes les policies workload et `tier-required` couvrent aussi les conteneurs éphémères. Vérifié en dry-run sur le cluster.
- **Leçon** : une policy d'admission se teste contre toutes les façons de faire tourner du code, pas seulement la création de pod.

### Le cluster ne se relève pas d'un nœud arrêté

- **Symptôme** : après `docker stop` puis `start` du nœud kind, le contrôleur de rapports Kyverno boucle en crash, et Argo CD garde certaines Applications dégradées alors que leurs enfants sont sains.
- **Cause** : OpenBao revient scellé, et le contrôleur de rapports, sous sa limite mémoire de 128 Mi, perd son bail de leader.
- **Correctif** : limite mémoire relevée ; la reprise documentée est la reconstruction complète (`task down && task up`). Le README a été corrigé : il affirmait à tort que `task up` faisait converger un nœud arrêté.
- **Leçon** : écrire ce que la plateforme ne sait **pas** faire est aussi important que le reste. La tolérance au redémarrage est dans la [roadmap](limites.md).

### Kyverno bloqué 12 minutes au démarrage

- **Symptôme** : sur un cluster neuf, les pods Kyverno restent bloqués environ 12 minutes sur le montage d'une ConfigMap.
- **Cause** : la ConfigMap de la CA locale est produite par trust-manager, créée plus tard que Kyverno.
- **Correctif** : ordre des vagues Argo CD revu, contrôles de santé sur les policies Kyverno, labels Pod Security posés de façon déclarative. Kyverno attend encore environ 2 minutes 30.
- **Leçon** : dans un bootstrap GitOps, l'ordre de création compte autant que le contenu. D'où l'invariant « aucun workload avant l'admission ».

## Bugs amont contournés

| Amont | Symptôme | Contournement | Condition de sortie |
|---|---|---|---|
| `kyverno test` (kyverno/sdk#125) | La CLI se bloque quand deux policies de vérification d'image évaluent la même ressource | Une suite de tests par policy | Le correctif kyverno/sdk#127 publié dans une CLI |
| Dependency-Track (#6132, #6957) | Aucune faille Debian rapprochée des paquets binaires du SBOM | La conversion VEX vise le paquet binaire via le qualificatif `upstream=` | Dependency-Track gère le lien source → binaire |
| Dependency-Track (#7094) | Pas d'import OpenVEX | Conversion OpenVEX → CycloneDX isolée dans un module | Import OpenVEX natif |
| CycloneDX (specification#609) | Pas d'équivalent pour la justification « non contrôlable par un attaquant » | Le motif est gardé dans le champ texte libre | La spécification ajoute la valeur |
| Adaptateur `github-ghcr` de Harbor | Un filtre « tout ce propriétaire » ne réplique rien : l'adaptateur ne sait pas lister GHCR | Chaque dépôt nommé explicitement, filtre golden dérivé du catalogue | L'adaptateur sait lister un propriétaire GHCR |
| Chart Harbor | Le type de cache proxy `quay` est absent de la liste du chart | Variable d'environnement qui remplace la liste | Le chart inclut `quay` |
| Stockage Kubescape | La liste des `ContainerProfile` renvoie les métadonnées sans le `spec` | Argo CD ignore `/spec` ; une annotation de hash sert de déclencheur de synchronisation | Kubescape documente ou supprime cette exigence |
| node-agent Kubescape | Le premier shell peut arriver avant le chargement du profil : pas d'alerte | La démo attend la ligne de log de chargement et relance un shell si besoin ; profil écrit à la main adopté immédiatement | node-agent expose un signal de disponibilité par pod |
| Gateway API de Cilium | Une restriction de port sur une règle egress vers un backend routé par la Gateway produit des 403 | Règle egress sans restriction de port pour ce seul saut | Cilium conserve le port de destination lors du contrôle |
| Argo CD sous WSL2 | Panique « counter cannot decrease » quand l'horloge de l'hôte recule | `ServerSideApply=true` sur la plupart des Applications ; resynchronisation manuelle occasionnelle | Correctif amont, ou un hôte dont l'horloge ne recule pas |
| GitHub Actions | Les PR ouvertes avec `GITHUB_TOKEN` ne déclenchent aucune CI | Aucun check obligatoire ; le mainteneur relance la CI à la main | PR ouvertes avec un jeton de GitHub App |
| Policy Reporter | L'API de l'UI ne renvoie pas la source de chaque résultat | Les tests filtrent avec `sources=` | L'API expose le champ |

## Choix de configuration assumés

- **`tomcat.version` forcé dans `hello-java`** au-dessus de la version gérée par Spring Boot, pour fermer une CVE. Renovate le suit avec une règle dédiée. À retirer quand Spring Boot embarque la version corrigée.
- **`config.preserve: false` sur Kyverno** : le hook post-suppression du chart empêche Argo CD d'élaguer l'Application. Kyverno est toujours installé et retiré par Argo CD ici, rien ne dépend de cette configuration après désinstallation.
- **Limites mémoire serrées puis desserrées** : VictoriaLogs a été tué à 160 Mi, une limite fixée quand le budget visait 10 Gio et gardée après le passage à 12 Gio. Grafana a été relevé à 768 Mi après un OOM dans Explore. Leçon : une limite posée sous contrainte doit être remesurée quand la contrainte change.
