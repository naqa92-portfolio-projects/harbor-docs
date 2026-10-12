# 0002 — Écrire toutes les policies d'admission en Kyverno CEL

**En une phrase** : les policies historiques de Kyverno (`ClusterPolicy`) disparaissent en 1.20 ; toutes les policies sont donc écrites dans les nouveaux types CEL, alignés sur l'admission native de Kubernetes.

| Statut | Date | Portée |
|---|---|---|
| Acceptée | septembre 2026 | Toutes les policies sous `policies/` |

## Contexte

L'admission doit imposer la chaîne de confiance des images (signature et attestations du workflow de build, base au catalogue lue dans la provenance, labels, Harbor comme seul registre) et Pod Security `restricted` dans les namespaces workload. Elle doit signaler les écarts dans les namespaces plateforme, et être **testable hors ligne en CI**.

Kyverno 1.19 propose deux familles : les `ClusterPolicy`/`Policy` historiques (dépréciées, supprimées en 1.20) et les types CEL (`ValidatingPolicy`, `ImageValidatingPolicy`, `MutatingPolicy`), alignés sur le `ValidatingAdmissionPolicy` de Kubernetes. L'autre candidat était OPA Gatekeeper avec Rego.

!!! tip "CEL en deux mots"
    *Common Expression Language* : le petit langage d'expressions que Kubernetes utilise nativement pour ses règles d'admission. Une policy CEL se lit comme une condition booléenne sur l'objet soumis.

## Décision

- Toutes les policies utilisent un type CEL. Aucune `ClusterPolicy` ni `Policy` dans le dépôt.
    - Workload (bloquantes) : `workload-image-signature`, `workload-golden-base`, `workload-registry`, `workload-image-labels`.
    - Workload (avertissement) : `workload-golden-base-deprecated`.
    - Workload (mutation) : `workload-pod-security`, qui pose le label Pod Security `restricted` sur les namespaces workload.
    - Niveaux de confiance : `tier-label` (seuls les admins posent le niveau) et `tier-required` (tout pod hors niveau valide est refusé).
    - Plateforme (audit) : `platform-registry-allow-list`, `platform-vendor-signatures`.
- Pod Security `restricted` est le **Pod Security Admission natif**, pas réimplémenté en policy.
- Les policies lisent leurs paramètres dans des ConfigMaps générées (catalogue golden, registres autorisés) : changer le catalogue ne demande aucune modification de policy.
- Les images admises sont réécrites en digest (`mutateDigest`).
- Les policies bloquantes tournent en `failurePolicy: Fail` : si Kyverno ne répond pas, le pod est refusé.
- Tests : `kyverno test` (une suite par policy) et Chainsaw sur un cluster kind éphémère, à chaque pull request.

## Alternatives écartées

- **`ClusterPolicy` historique** : supprimée en Kyverno 1.20, ce serait de la dette dès le premier jour.
- **OPA Gatekeeper + Rego** : très répandu dans les banques, mais la vérification de signature demande un fournisseur de données externe (Ratify) et Rego est un langage de plus à maîtriser. Une comparaison côte à côte reste dans la [roadmap](../limites.md#roadmap).

## Compromis et coût

- Les policies CEL rapportent `fail`, jamais `warn` : le cas « base dépréciée » apparaît en `fail` dans les PolicyReports alors que l'admission affiche un simple avertissement.
- Un refus par une policy CEL n'émet aucun événement Kubernetes : les violations sont comptées depuis l'audit log de l'API server.
- `kyverno test` se bloque quand deux policies de vérification d'image évaluent la même ressource (bug amont kyverno/sdk#125), d'où une suite de tests par policy.

## Ce qui me ferait changer d'avis

Un client qui a déjà Gatekeeper partout : je ferais la comparaison prévue avant d'imposer un second moteur.

## Dans le code

- [`policies`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/policies)
- [`tests/kyverno`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/tests/kyverno)
- [`tests/chainsaw`](https://github.com/naqa92-portfolio-projects/harbor-lab/tree/main/tests/chainsaw)
- [`.github/workflows/policies.yml`](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/.github/workflows/policies.yml)
