# Prochaines étapes

Le périmètre de la première version, et ce que je ferais ensuite. Chaque point dit pourquoi il compte et dans quelle direction aller.

## Périmètre de la première version

- **Démonstrateur sur un seul nœud** : un cluster kind, une instance par base de données. Le sujet est la chaîne de gouvernance ; la haute disponibilité relève de l'infrastructure qui l'hébergerait.
- **Reprise par reconstruction** : après un arrêt du nœud, la reprise documentée est `task down && task up` (environ 15 minutes).
- **Signature keyless publique** : adaptée à un dépôt public ; en environnement régulé, une clé KMS/HSM ou un Sigstore privé prend le relais sans changer les policies au-delà de l'attesteur ([ADR 0007](decisions/0007-signature-keyless.md)).
- **Namespaces de bootstrap exclus de l'admission**, par conception, pour éviter toute boucle de dépendance ([modèle de menace](modele-de-menace.md#namespaces-de-bootstrap-une-exclusion-assumee)).

## Améliorations identifiées

| Sujet | Pourquoi | Direction |
|---|---|---|
| CI sur les PR de ré-épinglage | Les PR ouvertes par les workflows avec `GITHUB_TOKEN` ne déclenchent pas de CI | Les ouvrir avec un jeton de GitHub App, puis rendre les checks obligatoires |
| Publication par digest | Publier l'image seulement après scan et signature simplifie les relances | Pousser par digest une fois le scan et la signature validés |
| Livraison de `dt-bridge` | Aujourd'hui l'état des VEX transmis est en mémoire et la file est bornée à 200 SBOM | Persister l'état, réessayer avec backoff, s'appuyer sur la relivraison Harbor |
| Descellement automatique d'OpenBao | Permettrait de reprendre après un arrêt du nœud sans reconstruction | Descellement Transit ou statique, redémarrages des contrôleurs conditionnés à leur disponibilité |
| Résultats Policy Reporter | Un résultat `error` apparaît par namespace pour les policies d'image ; la base dépréciée s'affiche en `fail` | Identifier la paire policy/ressource en cause ; aligner le rapport sur l'issue de l'admission |
| Tableau de bord | Titres tronqués, alertes runtime à filtrer, anciennes versions à archiver dans Dependency-Track | Passe de mise en page et de filtrage |
| Signatures éditeur à l'admission | Cilium et Argo CD tournent dans des namespaces exclus | Vérifier leurs images dès le bootstrap, ou les sortir du niveau exclu une fois Kyverno en place |

## Roadmap

| Sujet | Pourquoi | Direction |
|---|---|---|
| Bundle air-gap | Les sites régulés tournent souvent déconnectés : images, signatures, SBOM, VEX, racine Sigstore et bases de vulnérabilités doivent être importés | Un bundle OCI par release, importé dans Harbor et vérifié avant usage |
| Comparaison Gatekeeper / Rego | Beaucoup de banques utilisent déjà OPA Gatekeeper | Réécrire les règles workload en Rego (avec Ratify pour les signatures) et comparer expressivité, outillage de test et coût |
| Builds Buildah | Builds sans démon et sans root, courants là où Docker est interdit sur les agents | Variante de `build-image.yml` avec les mêmes étapes SBOM, provenance, scan et signature |
| Charts relocalisés en OCI | Étendre la gouvernance Harbor aux charts Helm | Relocaliser chaque chart dans Harbor (signé, scanné, immuable) |

## Évolutions amont suivies

- **Dependency-Track #6132 et #6957** : rapprochement des avis Debian (paquets source) avec les paquets binaires du SBOM. Rendra visible, faille par faille, l'effet des VEX DHI.
- **kyverno/sdk#125** (correctif #127) : permettra de regrouper les suites `kyverno test`.
- **Dependency-Track #7094** : import OpenVEX natif ; le module de conversion de `dt-bridge` pourra être retiré.
