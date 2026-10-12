# Retours d'expérience

Les problèmes rencontrés en construisant la plateforme, et comment ils ont été traités. Chaque entrée suit le même fil : **détecté → compris → corrigé → garde-fou**. Quand le correctif dépend d'un projet amont, la condition pour retirer le contournement est écrite : le jour où l'amont corrige, on sait quoi enlever.

## Cas marquants

### Dérive des filtres de réplication invisible pour OpenTofu

- **Détecté** : un filtre de réplication modifié dans l'UI Harbor n'apparaissait pas dans `tofu plan`.
- **Compris** : le provider `goharbor/harbor` ne relit pas tous les attributs de filtre dans son état (goharbor/terraform-provider-harbor#422). Avec un seul bloc `filters`, l'attribut autre que `name` est même perdu.
- **Corrigé** : deux blocs `filters` séparés (un `name`, un `tag`) ; `tofu apply -replace` pour toute modification faite hors du code.
- **Garde-fou** : l'UI Harbor est en lecture seule par convention, et le contournement est documenté avec sa condition de sortie.
- **Ce que j'en retiens** : « déclaratif » ne garantit pas la détection de toute dérive ; il faut savoir ce que le provider relit vraiment.

### Digest différent entre le cache proxy et l'amont

- **Détecté** : une image tirée via le cache Harbor apparaissait plusieurs minutes plus tard, sous un digest différent de Docker Hub.
- **Compris** : en lisant le code de Harbor (`manifestcache.go`) : pour un index multi-architecture tiré pour une seule plateforme, Harbor attend les autres plateformes puis pousse un index réduit, avec un nouveau digest.
- **Corrigé** : les contrôles du cache vérifient le digest du manifeste de la plateforme, pas le tag, et tolèrent ce délai.
- **Ce que j'en retiens** : lire le code amont tranche plus vite qu'empiler des hypothèses.

### Couverture des conteneurs éphémères par l'admission

- **Détecté** : en revue de sécurité, avant la livraison, un conteneur éphémère ajouté par `kubectl debug` n'était pas évalué par les policies workload.
- **Compris** : les policies ciblaient `pods`, pas la sous-ressource `pods/ephemeralcontainers`.
- **Corrigé** : toutes les policies workload et `tier-required` couvrent aussi les conteneurs éphémères, vérifié en dry-run sur le cluster.
- **Garde-fou** : un scénario Chainsaw de régression vérifie qu'un conteneur éphémère non conforme est refusé.
- **Ce que j'en retiens** : une policy d'admission se teste contre toutes les façons de lancer du code, pas seulement la création de pod.

### Reprise après arrêt du nœud

- **Détecté** : en revue, en arrêtant puis relançant le nœud kind : le contrôleur de rapports Kyverno ne reprenait pas, et OpenBao revenait scellé.
- **Compris** : sous sa limite mémoire de 128 Mi, le contrôleur perdait son bail de leader ; OpenBao, lui, se rescelle par conception à chaque redémarrage.
- **Corrigé** : limite mémoire relevée ; la procédure de reprise documentée est la reconstruction (`task down && task up`, environ 15 minutes), et le README a été aligné sur ce comportement.
- **Suite** : le descellement automatique d'OpenBao est dans les [prochaines étapes](limites.md).

### Ordre de démarrage entre trust-manager et Kyverno

- **Détecté** : sur un cluster neuf, les pods Kyverno attendaient environ 12 minutes le montage d'une ConfigMap.
- **Compris** : la ConfigMap de la CA locale est produite par trust-manager, créé après Kyverno.
- **Corrigé** : ordre des vagues Argo CD revu, contrôles de santé sur les policies Kyverno, labels Pod Security posés de façon déclarative. L'attente est tombée à environ 2 minutes 30.
- **Garde-fou** : l'invariant « aucun workload avant l'admission » est porté par l'ordre des vagues.

## Contournements de limites amont

| Amont | Constat | Contournement en place | Condition de retrait |
|---|---|---|---|
| `kyverno test` (kyverno/sdk#125) | La CLI se bloque quand deux policies de vérification d'image évaluent la même ressource | Une suite de tests par policy | Correctif kyverno/sdk#127 publié dans une CLI |
| Dependency-Track (#6132, #6957) | Les avis Debian (paquets source) ne sont pas rapprochés des paquets binaires du SBOM | La conversion VEX vise le paquet binaire via le qualificatif `upstream=` | Dependency-Track gère le lien source → binaire |
| Dependency-Track (#7094) | Pas d'import OpenVEX | Conversion OpenVEX → CycloneDX isolée dans un module | Import OpenVEX natif |
| CycloneDX (specification#609) | Pas d'équivalent pour la justification « non contrôlable par un attaquant » | Le motif est conservé dans le champ texte libre | La spécification ajoute la valeur |
| Adaptateur `github-ghcr` de Harbor | L'adaptateur ne sait pas lister un propriétaire GHCR | Chaque dépôt est nommé explicitement, filtre golden dérivé du catalogue | L'adaptateur sait lister un propriétaire |
| Chart Harbor | Le type de cache proxy `quay` est absent de la liste du chart | Variable d'environnement qui complète la liste | Le chart inclut `quay` |
| Stockage Kubescape | La liste des `ContainerProfile` ne renvoie pas le `spec` | Argo CD ignore `/spec` ; une annotation de hash déclenche la synchronisation | Kubescape documente ou supprime cette exigence |
| node-agent Kubescape | Le profil d'un conteneur neuf peut être chargé après son démarrage | La démo attend le chargement du profil ; un profil écrit à la main est adopté immédiatement | node-agent expose un signal de disponibilité par pod |
| Gateway API de Cilium | Une restriction de port sur une règle egress vers un backend routé par la Gateway produit des 403 | Règle egress sans restriction de port pour ce seul saut | Cilium conserve le port de destination lors du contrôle |
| Argo CD sous WSL2 | Panique « counter cannot decrease » quand l'horloge de l'hôte recule | `ServerSideApply=true` sur la plupart des Applications | Correctif amont |
| GitHub Actions | Les PR ouvertes avec `GITHUB_TOKEN` ne déclenchent pas de CI | Le mainteneur relance la CI sur la branche | PR ouvertes avec un jeton de GitHub App |
| Policy Reporter | L'API de l'UI ne renvoie pas la source de chaque résultat | Les tests filtrent avec `sources=` | L'API expose le champ |

## Choix de configuration documentés

- **`tomcat.version` forcé dans `hello-java`** au-dessus de la version gérée par Spring Boot, pour fermer une CVE. Renovate le suit avec une règle dédiée ; à retirer quand Spring Boot embarque la version corrigée.
- **`config.preserve: false` sur Kyverno** : le hook post-suppression du chart empêche Argo CD d'élaguer l'Application ; ici Kyverno est toujours installé et retiré par Argo CD.
- **Limites mémoire remesurées** : VictoriaLogs (160 Mi) et Grafana (768 Mi aujourd'hui) ont été réajustés quand le budget global est passé de 10 à 12 Gio. Une limite fixée sous contrainte se remesure quand la contrainte change.
