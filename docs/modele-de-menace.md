# Modèle de menace

harbor-lab décide quelles images de conteneurs peuvent tourner sur le cluster. Cette page dit **ce que l'admission vérifie**, niveau de confiance par niveau de confiance, et **les exceptions** sur lesquelles cette confiance repose.

## Actifs, menaces, contrôles

| Ce qu'on protège | Menace | Contrôle |
|---|---|---|
| Images des workloads | Image non signée, modifiée, ou construite ailleurs | Policy `workload-image-signature` : signature keyless et attestations CycloneDX, SPDX et SLSA v1 émises par le workflow `build-image.yml`, **et** provenance qui nomme ce dépôt comme appelant |
| Images de base | Image construite sur une base inconnue ou en fin de vie | Policy `workload-golden-base` : le digest de base lu dans la provenance vérifiée doit être `supported` ou `deprecated` dans le catalogue (`deprecated` produit un avertissement) |
| Chemin du registre | Image tirée hors des projets Harbor gouvernés | Policy `workload-registry` : uniquement `harbor…/golden/` et `harbor…/apps/` |
| Traçabilité | Image sans source, révision ou propriétaire | Policy `workload-image-labels` : labels OCI et `io.harborlab.*` obligatoires |
| Nœud | Conteneur privilégié ou root | Pod Security Admission natif, profil `restricted` imposé à chaque namespace workload |
| Dérive de tag | Un tag redirigé vers une autre image après admission | L'admission réécrit chaque image admise en digest (`mutateDigest`) |

## Niveaux de confiance

| Niveau | Namespaces | Admission |
|---|---|---|
| Workload | `harborlab.io/tier=workload` | Chaîne complète, bloquante (`Deny`) |
| Plateforme | `harborlab.io/tier=platform` | Signalé seulement (`Audit`) : registre autorisé, signatures éditeur de Kyverno, Cilium et Argo CD |
| Bootstrap | `kube-system`, `cilium`, `argocd`, `kyverno` | Aucune : exclus nommément |
| Aucun | tout autre namespace | Tout pod refusé (`tier-required`) : l'application des règles échoue fermée |

Seuls les administrateurs de la plateforme (cluster-admin et Argo CD, qui synchronise les labels depuis Git) choisissent un niveau. Quelqu'un qui peut modifier un namespace peut le placer en `workload`, mais ne peut ni retirer le label, ni le changer, ni choisir `platform` (policy `tier-label`).

!!! note "À noter"
    Les règles de signature éditeur pour Cilium et Argo CD ne s'appliquent que si ces images tournent dans un namespace `platform`. Ici, elles tournent uniquement dans les namespaces exclus `cilium` et `argocd`. En pratique, seules les images Kyverno (Policy Reporter par exemple) rencontrent cette règle.

## Namespaces de bootstrap : une exclusion assumée

`kube-system`, `cilium`, `argocd` et `kyverno` sont exclus nommément des policies, même s'ils portent un label. Ils font tourner ce dont l'admission elle-même dépend : CoreDNS et les add-ons de l'API server, le CNI, le contrôleur GitOps qui déploie les policies, et le moteur de policies.

Leur appliquer les policies crée une **boucle de dépendance** : une panne de Kyverno, de Rekor ou du registre, ou une erreur dans une policy, bloquerait les pods nécessaires pour réparer, et un redémarrage du cluster ne pourrait jamais converger.

C'est un risque documenté et accepté :

- Quelqu'un qui peut créer des pods dans ces namespaces contourne l'admission. L'accès est limité aux administrateurs du cluster et aux contrôleurs installés là ; aucun workload n'y tourne.
- Leurs images viennent de charts Helm et de valeurs versionnées et relues dans Git, pas de choix faits à l'exécution.

## Signature keyless sur le Rekor public

Les images sont signées **sans clé** : GitHub Actions obtient un certificat Fulcio de courte durée lié à l'identité du workflow (`build-image.yml` sur la branche `main` de ce dépôt), et chaque signature est inscrite dans le journal de transparence public Rekor. Aucune clé privée à voler ou à faire tourner, et chaque signature est auditable.

Les compromis, pour un environnement régulé comme une banque :

- La signature dépend de l'infrastructure publique Sigstore (Fulcio, Rekor, racine TUF), joignable depuis la CI et depuis l'admission.
- Le journal publie l'identité, le dépôt et le digest : acceptable pour un dépôt public, pas pour des métadonnées de build confidentielles.
- La confiance repose sur l'émetteur OIDC de GitHub et sur **le droit d'écriture** dans le dépôt. Le ruleset de `main` bloque la suppression et le force-push, mais ne restreint pas qui peut pousser : l'identité de signature est donc liée à quiconque a ce droit (ici, le seul propriétaire).

L'alternative bancaire est une signature par clé gardée dans un KMS ou un HSM (`cosign --key <kms-uri>` supporte AWS KMS, GCP KMS, Azure Key Vault et Vault / OpenBao Transit), éventuellement avec une instance Sigstore privée. Seule la définition de l'attesteur dans les policies change.

## Confiance propre à chaque environnement

Les policies commitées font confiance à `build-image.yml` exécuté depuis `refs/heads/main` uniquement. Une plateforme déployée depuis une autre révision (une branche de travail) doit aussi admettre les images que cette branche a construites. Elle fait donc confiance à **exactement une ref supplémentaire**, échappée en expression régulière, et seulement dans cet environnement-là. Jamais un motif comme `prd-.+` : une autre branche du dépôt reste non fiable.

Le revers : après un merge, les images épinglées ont été signées sur la branche de travail, que `main` ne reconnaît plus. Le workflow `release-repin.yml` les ré-épingle dans l'ordre des dépendances (Dockerfiles, puis workloads, puis fixtures, puis paramètres de test), une pull request par étape, sans jamais pousser directement sur `main`.

**Ce qu'il ne faut pas faire** : élargir l'identité commitée à `main` ou `prd-*` pour éviter ce ré-épinglage. N'importe quelle branche pourrait alors signer des images admissibles sur l'environnement que tous les consommateurs considèrent comme fiable par défaut.

## Constats connus

| Constat | Risque | Mitigation |
|---|---|---|
| Signature keyless sur le Rekor public plutôt qu'un KMS | Dépendance aux services publics Sigstore ; métadonnées de signature publiques | Documenté ci-dessus ; une banque signe avec une clé KMS/HSM ou un Sigstore privé |
| Le droit d'écriture ancre l'identité de signature | Quiconque peut pousser sur `main` peut signer des images admissibles | Une banque ajoute des revues obligatoires, des checks obligatoires et restreint qui peut pousser |
| OpenTofu ne voit pas toutes les modifications faites à la main dans Harbor | Un filtre de réplication modifié dans l'UI passe inaperçu au `plan` | L'UI Harbor est en lecture seule par convention ; `tofu apply -replace` restaure la règle |
| `dt-bridge` lit les attestations depuis Harbor | Un referrer falsifié tromperait le tri (pas l'admission, qui vérifie elle-même) | **Corrigé** : `dt-bridge` vérifie les bundles Sigstore contre l'identité de build et le dépôt appelant |
| La CA locale manquait d'extension `keyUsage` | Les clients X.509 stricts la refusaient ; `dt-bridge` relâchait la vérification Python | **Corrigé** : la CA porte un `keyUsage` critique, la vérification stricte reste active |
| Couverture des conteneurs éphémères (`kubectl debug`) | Un conteneur éphémère doit passer par les mêmes policies qu'un pod | **Corrigé** : les policies couvrent aussi `pods/ephemeralcontainers` |
