# Sécuriser Kubernetes avec Kubescape 4.0

Kubescape 4.0 est une plateforme de sécurité Kubernetes open-source qui scanne vos clusters, manifests et images pour détecter les vulnérabilités CVE et les mauvaises configurations. Cette version majeure introduit la correction automatique des manifests, le scan d'images, les Validating Admission Policies natives Kubernetes et le patch d'images vulnérables.

---

## Ce que vous allez apprendre

- Scanner un cluster selon les frameworks NSA, MITRE, CIS
- Détecter les CVE dans vos images conteneurs
- Corriger automatiquement les manifests avec `kubescape fix`
- Générer des Validating Admission Policies (VAP)
- Patcher les images vulnérables

> Toutes les commandes et exemples sont basés sur la version 4.0.1 de Kubescape, sortie en février 2026.

---

## Nouveautés de la version 4.0

La version 4.0 de Kubescape apporte des fonctionnalités majeures :

| Fonctionnalité | Description | Commande |
|----------------|-------------|----------|
| Scan d'images | Détection des CVE dans les images conteneurs | `kubescape scan image` |
| Fix automatique | Correction des manifests YAML insécurisés | `kubescape fix` |
| VAP natives | Génération de Validating Admission Policies CEL | `kubescape vap` |
| Patch d'images | Correction automatique des vulnérabilités d'images | `kubescape patch` |
| Serveur MCP | Intégration avec assistants IA | `kubescape mcpserver` |
| Nouveaux CIS | Support CIS v1.10, v1.12, AKS/EKS t1.8 | `kubescape list frameworks` |
| Performance | Optimisation CPU/mémoire pour scans intensifs | — |

---

## Fonctionnalités de Kubescape

Kubescape propose un ensemble riche de fonctionnalités pour renforcer la sécurité des environnements Kubernetes :

1. **Shift-left security** : Kubescape permet aux développeurs d'identifier les mauvais paramétrages dès la soumission des fichiers de manifeste, favorisant une approche proactive de la sécurité.

2. **Intégration IDE et CI/CD** : L'outil s'intègre facilement à des environnements de développement comme VSCode, Lens et à des plateformes CI/CD telles que GitHub et GitLab.

3. **Scan de clusters** : Kubescape peut scanner les clusters Kubernetes actifs à la recherche de vulnérabilités, de configurations incorrectes ou d'autres problèmes de sécurité.

4. **Support de plusieurs frameworks** : Il teste les configurations de sécurité selon divers cadres tels que la NSA, MITRE, SOC2 et d'autres.

5. **Validation des YAML et des charts Helm** : L'outil vérifie les fichiers YAML et les Helm charts selon les critères des cadres de sécurité, même en l'absence d'un cluster actif.

6. **Renforcement de Kubernetes** : Il permet d'identifier et de remédier aux mauvaises configurations et vulnérabilités via des scans manuels, récurrents ou déclenchés par un événement.

7. **Sécurité à l'exécution** : Kubescape offre une protection continue à l'exécution avec une surveillance constante et la détection des menaces.

8. **Gestion de la conformité** : L'outil aide à maintenir la conformité aux cadres et normes reconnus.

9. **Support multi-cloud** : Il offre une sécurité fluide sur plusieurs fournisseurs cloud et différentes distributions de Kubernetes.

---

## Installation

via le script officiel :

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
```

Ajoutez le binaire à votre PATH :

```bash
export PATH=$PATH:$HOME/.kubescape/bin
```

Vérifiez l'installation :

```bash
kubescape version
# Your current version is: 4.0.1
# Build date: 2026-02-12T15:17:22Z
```

### Docker

```bash
docker run -v ~/.kube:/root/.kube quay.io/kubescape/kubescape scan
```

### Opérateur Kubernetes

```bash
helm repo add kubescape https://kubescape.github.io/helm-charts/
helm repo update
helm upgrade --install kubescape kubescape/kubescape-operator \
  -n kubescape --create-namespace \
  --set clusterName=$(kubectl config current-context) \
  --set capabilities.continuousScan=enable
```

---

## CLI vs Opérateur : quel mode choisir ?

### CLI

**Cas d'usage** : scans ponctuels, pipelines CI/CD, analyse locale.

- Installation : binaire local (`curl`) ou container Docker
- Scan d'images : manuel avec `kubescape scan image`
- Registry privé : credentials passés en ligne de commande
- Rapports : fichiers locaux (SARIF, JSON, HTML, JUnit)
- Impact : aucun sur le cluster, exécution côté développeur/CI

```bash
# Exemple : scan CI avec seuil
kubescape scan . --compliance-threshold 80 --format sarif
```

### Opérateur

**Cas d'usage** : surveillance continue, inventaire complet des vulnérabilités.

- Installation : Helm chart dans le cluster
- Scan d'images : automatique sur les images déployées
- Registry privé : Secret Kubernetes référencé
- Rapports : CRDs Kubernetes, dashboard ARMO (optionnel)
- Impact : pods dans le namespace `kubescape`

> Utilisez le CLI pour les scans CI/CD et l'analyse locale. Déployez l'opérateur pour une surveillance continue.

---

## Fonctionnement de Kubescape

Kubescape utilise des contrôles basés sur les directives de sécurité fournies par la NSA, la CISA et Microsoft. Ces contrôles sont regroupés en cadres (frameworks) :

- **AllControls** : Exécute tous les contrôles disponibles
- **ArmoBest** : Recommandations spécifiques de l'équipe Kubescape
- **DevOpsBest** : Couverture basique des points essentiels
- **MITRE** : Conforme aux recommandations MITRE ATT&CK
- **NSA** : Conforme aux recommandations NSA/CISA
- **SOC2** : Axé sur la conformité SOC 2
- **CIS** : Contrôles spécifiques pour les versions Kubernetes AKS, EKS et standard

Pour afficher les frameworks disponibles :

```bash
kubescape list frameworks

╭──────────────────────╮
│ Supported frameworks │
├──────────────────────┤
│      AllControls     │
├──────────────────────┤
│       ArmoBest       │
├──────────────────────┤
│      DevOpsBest      │
├──────────────────────┤
│         MITRE        │
├──────────────────────┤
│          NSA         │
├──────────────────────┤
│         SOC2         │
├──────────────────────┤
│    cis-aks-t1.2.0    │
├──────────────────────┤
│    cis-aks-t1.8.0    │
├──────────────────────┤
│    cis-eks-t1.7.0    │
├──────────────────────┤
│    cis-eks-t1.8.0    │
├──────────────────────┤
│      cis-v1.10.0     │
├──────────────────────┤
│      cis-v1.12.0     │
╰──────────────────────╯
```

La v4.0 ajoute les frameworks CIS v1.10, CIS v1.12 et les versions t1.8 pour AKS/EKS.

---

## Nos Premiers scans avec kubescape

### Analyse d'un Cluster Kubernetes

Kubescape permet :

- D'identifier des nodes non sécurisés (clients Kubelet sans authentification TLS)
- De rechercher les interfaces exposées et non sécurisées
- De détecter si le cluster est affecté par des CVE connues
- De détecter les politiques réseau manquantes

Pour lancer une analyse :

```bash
kubescape scan
```

Exemple de sortie sur un cluster Minikube :

```
Security posture overview for cluster: 'minikube'

Control plane
╭────┬─────────────────────────────────────┬───────────────────────────────────────────╮
│    │ Control name                        │ Docs                                      │
├────┼─────────────────────────────────────┼───────────────────────────────────────────┤
│ ✅ │ API server insecure port is enabled │ https://kubescape.io/docs/controls/c-0005 │
│ ❌ │ Anonymous access enabled            │ https://kubescape.io/docs/controls/c-0262 │
│ ❌ │ Audit logs enabled                  │ https://kubescape.io/docs/controls/c-0067 │
│ ✅ │ RBAC enabled                        │ https://kubescape.io/docs/controls/c-0088 │
│ ❌ │ Secret/etcd encryption enabled      │ https://kubescape.io/docs/controls/c-0066 │
╰────┴─────────────────────────────────────┴───────────────────────────────────────────╯

Access control
╭────────────────────────────────────────────────────┬───────────┬────────────────────────────────────╮
│ Control name                                       │ Resources │ View details                       │
├────────────────────────────────────────────────────┼───────────┼────────────────────────────────────┤
│ Administrative Roles                               │     1     │ $ kubescape scan control C-0035 -v │
│ List Kubernetes secrets                            │     12    │ $ kubescape scan control C-0015 -v │
│ Minimize access to create pods                     │     4     │ $ kubescape scan control C-0188 -v │
│ Minimize wildcard use in Roles and ClusterRoles    │     1     │ $ kubescape scan control C-0187 -v │
│ Portforwarding privileges                          │     1     │ $ kubescape scan control C-0063 -v │
╰────────────────────────────────────────────────────┴───────────┴────────────────────────────────────╯

Secrets
╭─────────────────────────────────────────────────┬───────────┬────────────────────────────────────╮
│ Control name                                    │ Resources │ View details                       │
├─────────────────────────────────────────────────┼───────────┼────────────────────────────────────┤
│ Applications credentials in configuration files │     2     │ $ kubescape scan control C-0012 -v │
╰─────────────────────────────────────────────────┴───────────┴────────────────────────────────────╯

Network
╭────────────────────────┬───────────┬────────────────────────────────────╮
│ Control name           │ Resources │ View details                       │
├────────────────────────┼───────────┼────────────────────────────────────┤
│ Missing network policy │     16    │ $ kubescape scan control C-0260 -v │
╰────────────────────────┴───────────┴────────────────────────────────────╯

Workload
╭─────────────────────────┬───────────┬────────────────────────────────────╮
│ Control name            │ Resources │ View details                       │
├─────────────────────────┼───────────┼────────────────────────────────────┤
│ Host PID/IPC privileges │     0     │ $ kubescape scan control C-0038 -v │
│ HostNetwork access      │     0     │ $ kubescape scan control C-0041 -v │
│ HostPath mount          │     0     │ $ kubescape scan control C-0048 -v │
│ Non-root containers     │     15    │ $ kubescape scan control C-0013 -v │
│ Privileged container    │     0     │ $ kubescape scan control C-0057 -v │
╰─────────────────────────┴───────────┴────────────────────────────────────╯

Highest-stake workloads
───────────────────────

1. namespace: ingress-nginx, name: ingress-nginx-controller, kind: Deployment
2. namespace: default, name: my-app, kind: Deployment
3. namespace: kube-system, name: coredns, kind: Deployment

Compliance Score
────────────────

* MITRE: 63.90%
* NSA: 63.04%

View a full compliance report by running '$ kubescape scan framework nsa'
```

Pour un rapport HTML (avec liens de remédiation) :

```bash
kubescape scan --format html --output results.html
```

### Limiter le scan à un framework, un namespace

Scanner uniquement avec le framework NSA :

```bash
kubescape scan framework NSA
```

Scanner un namespace spécifique :

```bash
kubescape scan --include-namespaces default framework DevOpsBest
```

Exclure certains namespaces :

```bash
kubescape scan --exclude-namespaces kube-system,kube-public framework NSA
```

Pour découvrir toutes les options :

```bash
kubescape list --help
```

---

## Analyse des manifests YAML et des charts HELM

Kubescape permet d'analyser les manifests YAML et les Charts HELM, même directement dans un dépôt Git.

```bash
kubescape scan framework DevOpsBest https://github.com/kubescape/kubescape
```

Pour analyser des manifests YAML locaux :

```bash
kubescape scan *.yaml
```

Pour analyser un chart Helm :

```bash
kubescape scan https://github.com/AdminTurnedDevOps/PearsonCourses/tree/main/Helm-Charts-For-Kubernetes/Segment3/nginxupdate
```

Kubescape peut détecter :

- Services SSH non sécurisés fonctionnant à l'intérieur d'un conteneur
- Utilisation de sudo dans la commande de démarrage d'un conteneur
- Exécution de conteneurs en tant que root ou avec des capacités excessives
- Pods prenant trop de ressources CPU et Mémoire

À la fin de l'analyse kubescape fournit un score de risque de 0% (très sécurisé) à 100% (non sécurisé).

---

## Scan d'images conteneurs

Kubescape scanne les images conteneurs pour détecter les vulnérabilités CVE, sans avoir besoin d'un cluster actif (mode CLI).

> Les résultats dépendent de la base CVE utilisée (NVD, OSV, etc.) et de sa fraîcheur.

### Scanner une image

```bash
kubescape scan image nginx:1.27-alpine

╭──────────┬────────────────┬────────────┬───────────┬───────────┬───────────────────╮
│ Severity │ Vulnerability  │ Component  │ Version   │ Fixed in  │ Image             │
├──────────┼────────────────┼────────────┼───────────┼───────────┼───────────────────┤
│ Critical │ CVE-2025-15467 │ libcrypto3 │ 3.3.3-r0  │ 3.3.6-r0  │ nginx:1.27-alpine │
│ Critical │ CVE-2025-15467 │ libssl3    │ 3.3.3-r0  │ 3.3.6-r0  │ nginx:1.27-alpine │
...

81 vulnerabilities found
* 4 Critical
* 27 High
* 38 Medium
* 12 Other
```

### Authentification registry privé

```bash
kubescape scan image registry.example.com/app:v1.2 \
  --username $REGISTRY_USER \
  --password $REGISTRY_PASSWORD
```

Pour l'opérateur : configurez un Secret Kubernetes avec vos credentials.

### Scanner les images d'un workload

Combinez scan de configuration et scan d'images :

```bash
kubescape scan workload Deployment/my-app --namespace production --scan-images
```

---

## Correction automatique des manifests (v4.0)

La commande `kubescape fix` analyse un rapport de scan et corrige automatiquement les manifests YAML.

**Étape 1** : Scanner le manifest et générer un rapport JSON

```bash
kubescape scan deployment.yaml --format json --output scan-results.json
```

**Étape 2** : Appliquer les corrections automatiques

```bash
kubescape fix scan-results.json
```

Kubescape affiche les changements appliqués :

```
The following changes will be applied:
File: deployment.yaml
Resource: insecure-app
Kind: Deployment
Changes:
   1) spec.template.spec.containers[0].securityContext.runAsGroup = 1000
   2) spec.template.spec.containers[0].securityContext.readOnlyRootFilesystem = true
   3) spec.template.spec.containers[0].securityContext.allowPrivilegeEscalation = false
   4) spec.template.spec.containers[0].securityContext.privileged = false
   5) spec.template.spec.containers[0].securityContext.runAsNonRoot = true
```

**Étape 3** : Vérifier les changements (dry-run)

```bash
kubescape fix scan-results.json --dry-run
```

**Bonnes pratiques** :

- Toujours lancer `--dry-run` d'abord et relire les changements
- Exécuter `fix` dans une branche dédiée ou une PR, jamais sur `main`
- Versionner vos manifests avant correction (Git)
- Attention aux manifests générés : si vous utilisez Helm/Kustomize, `fix` modifie le rendu final, pas les sources

---

## Validating Admission Policies (v4.0)

Kubescape peut générer des Validating Admission Policies (VAP) natives Kubernetes pour bloquer les ressources non conformes avant leur création.

### Déployer la bibliothèque de policies CEL

```bash
kubescape vap deploy-library | kubectl apply -f -
```

Cette commande crée :
- Un CustomResourceDefinition `ControlConfiguration`
- Les ValidatingAdmissionPolicy pour chaque contrôle Kubescape

### Créer un binding de policy

Bloquer les conteneurs privilégiés dans le namespace `default` :

```bash
kubescape vap create-policy-binding \
  --name no-privileged \
  --policy c-0057 \
  --namespace=default | kubectl apply -f -
```

Résultat généré :

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-privileged
spec:
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: kubernetes.io/metadata.name
        operator: In
        values:
        - default
  policyName: c-0057
  validationActions:
  - Deny
```

| Version K8s | Statut | Notes |
|-------------|--------|-------|
| 1.26–1.27 | Alpha | Feature gate + API alpha à activer |
| 1.28–1.29 | Beta | Souvent désactivé par défaut |
| 1.30+ | GA | Activé par défaut |

---

## Patch d'images vulnérables

La commande `kubescape patch` s'appuie sur Copacetic et BuildKit pour reconstruire une image en mettant à jour les packages vulnérables.

### Prérequis

Le patching nécessite buildkitd en cours d'exécution :

```bash
sudo buildkitd &
```

### Patcher une image

```bash
sudo kubescape patch --image docker.io/library/nginx:1.22
```

Kubescape :
1. Scanne l'image pour identifier les CVE
2. Génère un Dockerfile temporaire avec les mises à jour
3. Reconstruit l'image avec le tag `-patched`

### Options utiles

```bash
# Spécifier un tag personnalisé
kubescape patch --image nginx:1.22 --tag nginx:1.22-secure

# Mode verbose pour voir le détail
kubescape patch --image nginx:1.22 --verbose

# Définir un seuil de sévérité (échoue si CVE Critical)
kubescape patch --image nginx:1.22 --severity-threshold Critical
```

> Le patch ne fonctionne que pour les packages système (apk, apt, yum). Les dépendances applicatives (npm, pip, go modules) ne sont pas patchées automatiquement.

---

## Utilisation dans les outils de CI/CD

Kubescape s'intègre dans la plupart des pipelines CI/CD :

- GitHub Actions (action officielle)
- GitLab CI
- Azure DevOps
- Jenkins
- ArgoCD

### Stratégie de gating recommandée

| Contexte | Comportement | Configuration |
|----------|--------------|---------------|
| Pull Request | Warning (non bloquant) | Score affiché, pas de fail |
| main/master | Bloquant | `--compliance-threshold 80 --severity-threshold Critical` |
| Release/tag | Bloquant strict | `--compliance-threshold 90 --severity-threshold High` |

### Pipeline GitHub Actions avec seuils

```yaml
name: Kubescape Security Scan
on: [push, pull_request]

jobs:
  security-scan:
    runs-on: ubuntu-24.04
    steps:
    - uses: actions/checkout@v4

    - name: Install Kubescape
      run: |
        curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
        echo "$HOME/.kubescape/bin" >> $GITHUB_PATH

    - name: Scan manifests
      run: |
        kubescape scan . \
          --format sarif \
          --output results.sarif \
          --compliance-threshold 80 \
          --severity-threshold High

    - name: Upload to GitHub Security
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: results.sarif
```

### Options de seuils pour la CI

| Option | Description | Exemple |
|--------|-------------|---------|
| `--compliance-threshold` | Score minimum de conformité (%) | `--compliance-threshold 80` |
| `--severity-threshold` | Échoue si vulnérabilités >= sévérité | `--severity-threshold High` |
| `--format` | Format de sortie (sarif, json, junit, html) | `--format sarif` |

Le format SARIF s'intègre nativement avec GitHub Security.

---

## Gestion des exceptions et baseline

### Créer une baseline

Figez l'état actuel comme référence :

```bash
kubescape scan --format json --output baseline.json
```

### Exclure des contrôles ou ressources

Créez un fichier `exceptions.json` :

```json
{
  "name": "Exceptions projet X",
  "policyType": "exceptions",
  "exceptions": [
    {
      "name": "Allow privileged for monitoring",
      "policyId": "C-0057",
      "resources": [
        {
          "designator": "Namespace=monitoring"
        }
      ]
    }
  ]
}
```

Appliquez lors du scan :

```bash
kubescape scan --exceptions exceptions.json
```

> Stockez `exceptions.json` dans Git, avec une revue obligatoire pour chaque modification.

---

## Utilisation de l'extension VSCode

Kubescape fournit une extension VSCode qui analyse les manifests en temps réel pendant l'édition.

L'extension détecte automatiquement :

- Les configurations non sécurisées
- Les images vulnérables
- Les manques de limites de ressources
- Les violations de bonnes pratiques

---

## À retenir

- Kubescape est une plateforme complète de sécurité Kubernetes
- CLI pour les scans ponctuels et CI/CD, Opérateur pour la surveillance continue
- La commande `scan` audite clusters, manifests et images selon NSA, MITRE, CIS
- Le scan d'images détecte les CVE (résultats dépendent de la fraîcheur de la base)
- `kubescape fix` corrige automatiquement les manifests (toujours `--dry-run` d'abord)
- Les VAP bloquent les déploiements non conformes (GA depuis K8s 1.30)
- `kubescape patch` utilise Copacetic/BuildKit pour corriger les images
- Gérez vos exceptions dans Git pour éviter le drift et la fatigue d'alertes
- Les seuils CI (`--compliance-threshold`, `--severity-threshold`) automatisent le contrôle qualité

---

## Ressources

- [Documentation officielle Kubescape](https://kubescape.io/docs/)
- [Repository GitHub](https://github.com/kubescape/kubescape)
- [Hub des contrôles](https://kubescape.io/docs/controls/)
- [Helm Charts pour l'opérateur](https://github.com/kubescape/helm-charts)
