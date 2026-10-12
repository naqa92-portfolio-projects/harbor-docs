# Démo

Neuf scénarios montrent la plateforme réagir. Chacun est une tâche du Taskfile, lancée sur une plateforme démarrée. Une tâche ne sort en succès que si elle **a observé** la réaction attendue, pas seulement soumis quelque chose. Les commandes exactes sont dans le [README de harbor-lab](https://github.com/naqa92-portfolio-projects/harbor-lab#readme) et le [Taskfile](https://github.com/naqa92-portfolio-projects/harbor-lab/blob/main/Taskfile.yml).

## Ce que chaque scénario prouve

### À l'admission : ce qui n'entre pas

| Scénario | Ce qu'on soumet | Réaction | Ce que ça prouve |
|---|---|---|---|
| `demo:unsigned` | Une image conforme, recopiée sans aucune signature | Refus par `workload-image-signature` | Rien n'entre sans signature |
| `demo:foreign-signer` | La même image, signée et attestée par **une autre identité** que le workflow de build | Même refus | Une signature valide de la mauvaise identité ne vaut rien |
| `demo:non-golden-base` | Une image construite directement sur une image DHI, hors catalogue | Refus par `workload-golden-base` | La base est lue dans la provenance signée, pas dans un label déclaratif |
| `demo:eol-base` | Une image construite sur Java 26, marqué `eol` au catalogue | Refus par `workload-golden-base` | Le cycle de vie des bases est appliqué automatiquement |
| `demo:direct-dockerhub` | `busybox` tiré directement de Docker Hub | Refus par `workload-registry` | Même si le nœud passe par le cache Harbor, la référence doit nommer Harbor |
| `demo:root` | Une image gouvernée, mais un pod qui tourne en root | Refus par Pod Security `restricted` | La chaîne d'image ne remplace pas la sécurité du pod |

![Refus d'une image signée par une autre identité](assets/demo/P1-02.png)

### À l'admission : ce qui entre, sous conditions

| Scénario | Ce qu'on soumet | Réaction | Ce que ça prouve |
|---|---|---|---|
| `demo:deprecated-base` | Une image construite sur Python 3.10, marqué `deprecated` | **Admise avec un avertissement**, image réécrite en digest | On peut laisser une équipe migrer sans la bloquer, tout en le rendant visible |

![Admission avec avertissement de base dépréciée](assets/demo/P1-07.png)

![La référence du pod admis est en digest](assets/demo/P1-08.png)

Lancé dans un namespace sans label de niveau, n'importe quel scénario est refusé par `tier-required` : c'est voulu, un namespace oublié ne doit jamais échapper aux contrôles.

### À l'exécution

| Scénario | Ce qu'on fait | Réaction | Ce que ça prouve |
|---|---|---|---|
| `demo:runtime-shell` | Ouvrir un shell dans un conteneur gouverné en cours d'exécution | Alerte Kubescape `Unexpected process launched` en quelques secondes, visible dans Grafana | La gouvernance ne s'arrête pas à l'admission |
| `demo:vex` | Rejouer l'événement Harbor d'une image golden | Dependency-Track reçoit le SBOM attesté et accepte le VEX de l'éditeur (`COMPLETED`) | Le tri se fait sur ce qui a été signé au build |

!!! note "À noter"
    Le scénario `demo:vex` prouve que le VEX est accepté et traité, **pas** que chaque faille Debian apparaît comme « non concernée ». Dependency-Track ne rapproche pas encore les paquets source et binaires Debian (évolution amont suivie). Voir [Prochaines étapes](limites.md).

## Parcours enregistrés

Trois parcours ont été enregistrés sur la plateforme, 20 Applications Argo CD synchronisées et saines.

### P1 — Voir la plateforme refuser une image non conforme

Les sept scénarios d'admission enchaînés dans un terminal.

<video controls width="100%" src="../assets/demo/P1.mp4"></video>

### P2 — Suivre une image du golden path jusqu'au tri des CVE

Dans Harbor, l'image `hello-java` est signée et porte quatre accessoires : la signature, deux SBOM et la provenance.

![Artefact hello-java signé dans Harbor](assets/demo/P2-03.png)

L'accessoire de provenance porte le type `https://slsa.dev/provenance/v1`. Harbor les affiche tous comme `signature.cosign`, une particularité d'affichage.

![Type réel de l'accessoire : provenance SLSA v1](assets/demo/P2-03c.png)

Dans Dependency-Track, le projet `hello-java` liste 1 312 composants, ceux du SBOM attesté au build.

![Composants du SBOM attesté dans Dependency-Track](assets/demo/P2-05.png)

Pour `golden/python`, aucune CVE Debian n'est listée : c'est l'évolution amont décrite plus haut, pas une absence de faille.

![Aucune CVE Debian listée pour golden/python](assets/demo/P2-fin.png)

### P3 — Repérer un shell dans un conteneur depuis Grafana

Le tableau de bord *image posture* montre les CVE par sévérité avant et après VEX, avec leur source, la part d'images signées et attestées en cours d'exécution, les refus d'admission et les alertes runtime.

![Tableau de bord image posture](assets/demo/P3-03.png)

Dans VictoriaLogs, l'événement Kubescape nomme le pod ciblé par la démo.

![Détail de l'alerte Kubescape dans Grafana Explore](assets/demo/P3-06h.png)

Dans Policy Reporter, la base dépréciée apparaît en `fail` et non en `warn` : un comportement des policies CEL, documenté dans l'[ADR 0002](decisions/0002-kyverno-cel.md).

![PolicyReport de la base dépréciée](assets/demo/P3-07d.png)
