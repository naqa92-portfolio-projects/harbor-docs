# Limites connues

Ce que la plateforme ne fait pas, ou fait imparfaitement. Je préfère le dire ici plutôt que de le découvrir en entretien.

## Ce que je ne prétends pas couvrir

- **Haute disponibilité** : un seul nœud kind, une instance par base de données. Le projet démontre une chaîne de gouvernance, pas une infrastructure de production.
- **Reprise après arrêt du nœud** : après un arrêt du nœud, OpenBao revient scellé et certains contrôleurs ne reconvergent pas seuls. La reprise est une reconstruction complète, environ 15 minutes.
- **Confiance en environnement régulé** : la signature keyless dépend du Sigstore public, et le droit d'écriture sur `main` fait partie de la chaîne de confiance. Une banque utiliserait une clé KMS/HSM et des revues obligatoires ([ADR 0007](decisions/0007-signature-keyless.md)).
- **Namespaces de bootstrap** : `kube-system`, `cilium`, `argocd` et `kyverno` sont exclus de l'admission, par conception ([modèle de menace](modele-de-menace.md#namespaces-de-bootstrap-une-exclusion-assumee)).
- **Signatures éditeur** : les règles de signature éditeur pour Cilium et Argo CD ne rencontrent jamais de vrai pod, puisque ces composants tournent dans des namespaces exclus.

## Dette après la première version

| Sujet | Pourquoi c'est important | Direction |
|---|---|---|
| Les PR de ré-épinglage ne lancent aucune CI | Ouvertes avec `GITHUB_TOKEN`, elles fusionnent des changements de digest sans vérification automatique | Les ouvrir avec un jeton de GitHub App, puis rendre les checks obligatoires |
| Tag non signé après un build échoué | Le push a lieu avant le scan et la signature ; un échec laisse un tag non signé qu'un nouveau passage ne re-signe pas | Pousser par digest seulement après scan et signature |
| Garanties de livraison de `dt-bridge` | État des VEX transmis en mémoire, pas de nouvelle tentative en cas d'échec, file limitée à 200 SBOM | Persister l'état, réessayer avec backoff, laisser Harbor relivrer |
| Résultats `error` dans Policy Reporter | Chaque namespace porte un résultat `error` sans cause visible | Identifier la paire policy/ressource fautive |
| Base dépréciée affichée en `fail` | Les rapports disent `fail` quand l'admission dit « avertissement » | Faire correspondre le rapport à l'issue de l'admission |
| Ergonomie du tableau de bord | Titres tronqués, alertes runtime dominées par le bruit, projets Dependency-Track noyés sous les versions | Passe de mise en page, filtrage des règles, désactivation des anciennes versions |
| Marge mémoire de Grafana | Explore sur VictoriaLogs monte à ~640 Mi pour une limite de 768 Mi | Remesurer après chaque mise à jour |

## Roadmap

| Sujet | Pourquoi | Direction |
|---|---|---|
| Bundle air-gap | Les sites régulés tournent souvent déconnectés : images, signatures, SBOM, VEX, racine Sigstore et bases de vulnérabilités doivent être importés | Un bundle OCI par release, importé dans Harbor et vérifié avant usage |
| Comparaison Gatekeeper / Rego | Beaucoup de banques ont déjà OPA Gatekeeper ; le choix de Kyverno doit être étayé | Réécrire les règles workload en Rego (avec Ratify pour les signatures) et comparer expressivité, outillage de test et coût |
| Builds Buildah | Builds sans démon et sans root, courants là où Docker est interdit sur les agents | Variante de `build-image.yml` avec les mêmes étapes SBOM, provenance, scan et signature |
| Charts relocalisés en OCI | Les charts Helm viennent de l'amont, hors gouvernance Harbor | Relocaliser chaque chart dans Harbor (signé, scanné, immuable) |
| Tolérance au redémarrage | Voir plus haut | Descellement automatique d'OpenBao, redémarrages des contrôleurs conditionnés à leur disponibilité |

## Problèmes amont suivis

- **Dependency-Track #6132 et #6957** : rapprochement des avis Debian (paquets source) avec les paquets binaires du SBOM. Débloquerait la preuve, faille par faille, de l'effet des VEX DHI.
- **kyverno/sdk#125** (correctif #127) : blocage de `kyverno test`. Permettrait de regrouper les suites de tests.
- **Dependency-Track #7094** : import OpenVEX natif. Supprimerait le module de conversion de `dt-bridge`.
