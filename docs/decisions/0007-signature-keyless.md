# 0007 — Signer sans clé avec Sigstore, identité liée à la branche `main` de ce dépôt

**En une phrase** : les images sont signées *keyless* avec l'identité OIDC du workflow GitHub, et seule la branche `main` de ce dépôt produit des images admissibles en production.

| Statut | Date | Portée |
|---|---|---|
| Acceptée, resserrée le 2026-09-28 | septembre 2026 | Signatures et attestations des images golden, applicatives et de test |

## Contexte

L'admission doit prouver qu'une image a été construite par le workflow de la plateforme, depuis ce dépôt. Signer avec une clé suppose une clé privée stockée, renouvelée et protégée quelque part (un KMS ou un HSM dans une banque) ; un dépôt de portfolio public n'a rien de tout ça.

La signature *keyless* de Sigstore lie un certificat Fulcio de courte durée à l'identité OIDC GitHub Actions du workflow, et inscrit chaque signature dans le journal de transparence Rekor.

## Décision

- `build-image.yml` signe et atteste en keyless avec cosign v3 : signature, SBOM CycloneDX, SBOM SPDX et provenance SLSA v1.
- L'admission vérifie l'émetteur `https://token.actions.githubusercontent.com` et l'identité `build-image.yml@refs/heads/main` **de ce dépôt uniquement**.
- **Liaison au dépôt** : `build-image.yml` est un workflow réutilisable public ; sans contrôle, son identité de signature est la même quel que soit le dépôt qui l'appelle. Le job de build ne tourne donc que si `github.repository` est ce dépôt, l'admission exige que la provenance vérifiée nomme ce dépôt comme appelant, et `dt-bridge` l'exige dans l'extension du certificat.
- **Signature du seul digest construit** : les étapes de signature ne signent que le digest que ce même job vient de construire, jamais un tag préexistant sur GHCR.
- **Clé DHI épinglée** : la signature de l'image de base DHI est vérifiée contre `.github/keys/dhi.pub`, commitée dans le dépôt, et non contre une clé téléchargée au moment du build.
- **Confiance propre à l'environnement** : une plateforme déployée depuis une branche de travail fait confiance à cette seule branche en plus de `main`, et seulement dans cet environnement (voir [modèle de menace](../modele-de-menace.md#confiance-propre-a-chaque-environnement)).

## Pourquoi la décision a été resserrée

La première version faisait confiance à `main` **ou à toute branche `prd-*`**. La revue a montré que n'importe quelle branche `prd-*` pouvait alors signer des images admissibles en production, ce qui est plus large que voulu. Depuis le 2026-09-28 : `main` seule par défaut, plus la liaison au dépôt, la signature du seul digest construit et la clé DHI épinglée.

## Alternatives écartées

- **Clé cosign classique dans un secret GitHub** : une clé longue durée à protéger et faire tourner, pour un gain nul ici.
- **Clé dans un KMS** : la bonne réponse en banque, mais aucun KMS dans un portfolio. Le passage ne change que la définition de l'attesteur dans les policies.

## Compromis et coût

- Plus de clé privée à gérer ; chaque signature est publique et auditable dans Rekor.
- Signature et vérification dépendent de l'infrastructure publique Sigstore (Fulcio, Rekor, racine TUF).
- **Le droit d'écriture sur `main` fait partie de la chaîne de confiance.** Le ruleset bloque suppression et force-push, pas qui peut pousser. Ici, c'est le seul propriétaire.
- Après chaque merge d'une branche de travail, les images épinglées doivent être ré-épinglées (`release-repin.yml`).

## Ce qui me ferait changer d'avis

Un contexte régulé : clé KMS/HSM ou Sigstore privé, revues obligatoires et restriction des pushs sur `main`.

## Dans le code

- [`.github/workflows/build-image.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/build-image.yml)
- [`policies/workload/image-signature.yaml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/policies/workload/image-signature.yaml)
- [`apps/dt-bridge`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/apps/dt-bridge)
- [`.github/keys/dhi.pub`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/keys/dhi.pub)
- [`.github/workflows/release-repin.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/release-repin.yml)
