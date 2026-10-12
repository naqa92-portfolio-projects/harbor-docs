# Questions d'entretien

Les questions qu'on me pose, ou qu'on devrait me poser, avec la réponse courte que je donne à l'oral. Chaque réponse renvoie à la page qui la détaille.

## Le projet

??? question "En une phrase, c'est quoi ?"
    Une plateforme qui garantit que seules des images construites, signées et attestées par la plateforme elle-même peuvent tourner sur le cluster, et qui suit leurs vulnérabilités jusqu'à l'exécution. Une équipe applicative n'écrit que `FROM <image golden>` et une ligne de workflow.

??? question "Pourquoi un seul cluster kind et pas un vrai cloud ?"
    Pour que n'importe qui puisse le relancer avec 16 Gio de RAM, sans compte cloud ni facture. Le sujet est la chaîne de gouvernance, pas l'infrastructure. Tout ce qui est démontré ici (signatures, admission, tri) se transpose tel quel sur un cluster managé.

??? question "Un point de sécurité non évident ?"
    La liaison de l'identité de signature au dépôt. Le workflow de build est réutilisable et public : sans contrôle, n'importe quel dépôt qui l'appelle obtiendrait la même identité de signature. L'admission exige donc que la provenance signée nomme ce dépôt comme appelant. Cette faille a été trouvée en revue du modèle de menace, puis corrigée. → [ADR 0007](decisions/0007-signature-keyless.md)

## Les choix techniques

??? question "Pourquoi Kyverno plutôt que OPA Gatekeeper ?"
    Kyverno vérifie les signatures d'images nativement, et ses policies CEL ressemblent à l'admission native de Kubernetes. Gatekeeper demande un fournisseur externe (Ratify) pour les signatures et un langage de plus, Rego. Comme beaucoup de banques ont déjà Gatekeeper, une comparaison côte à côte est dans la roadmap. → [ADR 0002](decisions/0002-kyverno-cel.md)

??? question "Pourquoi signer sans clé ? Une banque ne ferait pas ça."
    Exact, et c'est écrit. Dans un dépôt public sans KMS, une clé longue durée serait un secret de plus à protéger, pour rien. Keyless lie la signature à l'identité du workflow et la rend auditable dans Rekor. En banque, je passerais à une clé KMS ou HSM : seule la définition de l'attesteur change dans les policies. → [ADR 0007](decisions/0007-signature-keyless.md)

??? question "Pourquoi lire la base dans la provenance plutôt que dans un label de l'image ?"
    Un label est déclaratif : n'importe qui peut écrire ce qu'il veut dans un Dockerfile. La provenance SLSA est produite et signée par le workflow de build, qui a réellement vu la base utilisée. → [Architecture](architecture.md)

??? question "Pourquoi Kubescape plutôt que Falco ?"
    Budget mémoire. Falco ne couvre que le runtime ; il aurait fallu Trivy Operator en plus pour le scan et la posture. Kubescape fait les trois avec un seul agent. Je perds le grand catalogue de règles de Falco, ce qui est acceptable ici. → [ADR 0003](decisions/0003-kubescape.md)

??? question "Pourquoi écrire un service maison, dt-bridge ?"
    Dependency-Track n'a pas d'intégration Harbor et n'importe pas OpenVEX. Je voulais qu'il trie le SBOM signé au build, pas un SBOM régénéré. dt-bridge vérifie la signature, envoie le SBOM et convertit les VEX. Le code de conversion est isolé pour pouvoir être supprimé d'un bloc. → [ADR 0004](decisions/0004-dependency-track-dt-bridge.md)

??? question "Pourquoi OpenTofu pour configurer Harbor ?"
    harbor-cli affichait le secret des robots sur la sortie standard et n'était pas idempotente. Le provider officiel est déclaratif, les secrets passent en écriture seule et l'état est chiffré. Sa limite : il ne voit pas toutes les modifications faites dans l'UI. → [ADR 0005](decisions/0005-opentofu-goharbor.md)

??? question "Que se passe-t-il si Harbor tombe ?"
    Les pulls repartent vers l'amont grâce au miroir containerd transparent. Harbor n'est jamais nécessaire à son propre redémarrage. Les workloads, eux, référencent Harbor explicitement : ils attendent son retour. → [ADR 0006](decisions/0006-miroir-transparent-niveaux-confiance.md)

??? question "Pourquoi exclure kube-system, cilium, argocd et kyverno de l'admission ?"
    Ce sont les composants dont l'admission dépend. Si une erreur de policy ou une panne de Rekor bloquait leurs pods, le cluster ne pourrait plus se réparer. C'est un risque assumé et écrit, limité aux administrateurs. → [Modèle de menace](modele-de-menace.md#namespaces-de-bootstrap-une-exclusion-assumee)

??? question "Trivy et Grype ne donnent pas les mêmes CVE : lequel croire ?"
    Aucun des deux seul. Trivy est la barrière au build et dans Harbor, Grype est la vue du cluster, et le tri se fait dans Dependency-Track. Le tableau de bord étiquette chaque compte avec sa source et n'additionne jamais deux scanners. → [ADR 0008](decisions/0008-divergence-grype-trivy.md)

## Ce qui a cassé, ce que j'ai appris

??? question "Quel bug t'a le plus appris ?"
    Le cache proxy de Harbor qui renvoie un autre digest que Docker Hub. L'explication est venue du code de Harbor lui-même : pour un index multi-architecture, il attend les autres architectures, puis pousse un index réduit avec un nouveau digest. Leçon : lire le code amont plutôt qu'empiler des hypothèses. → [Ce qui a cassé](incidents.md)

??? question "Une faille trouvée en revue ?"
    `kubectl debug` permettait d'ajouter un conteneur éphémère qui échappait à l'admission : les policies ne couvraient pas la sous-ressource `pods/ephemeralcontainers`. Corrigé et testé. Une policy se teste contre toutes les façons de faire tourner du code. → [Ce qui a cassé](incidents.md#kubectl-debug-contournait-ladmission)

??? question "Qu'est-ce que tu changerais si tu recommençais ?"
    Je traiterais la reprise après redémarrage dès le début (descellement automatique d'OpenBao) au lieu de documenter une reconstruction complète. Et j'ouvrirais les PR automatiques avec un jeton de GitHub App dès le départ, pour que la CI tourne dessus. → [Limites connues](limites.md)

??? question "Comment sais-tu que ça marche ?"
    143 tests bats, 17 scénarios Chainsaw sur un cluster éphémère, 8 suites `kyverno test` et 26 tests pytest, plus neuf scénarios de démo qui ne réussissent que s'ils observent la réaction attendue. → [Démo](demo.md)
