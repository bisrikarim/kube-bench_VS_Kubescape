# Synthèse — kube-bench vs Kubescape

## En résumé

Les deux outils ne font pas la même chose. Ils sont **complémentaires** mais si on doit en choisir un seul pour notre contexte, c'est **Kubescape**.

## Résultats du POC

Tests réalisés sur un cluster k3s v1.29 (WSL2, Ubuntu 22.04) avec :
- 1 namespace `client-alpha` : workloads propres (SecurityContext, limits, probes)
- 1 namespace `bad-practices` : 6 pods volontairement non conformes (privileged, root, hostPath, hostNetwork, capabilities, no limits)

| Outil | Scan | Failed | Score | Détecte les bad pods ? |
|---|---|---|---|---|
| kube-bench | k3s-cis-1.24 (cluster) | 7 FAIL, 49 WARN | 42 PASS | **Non** — scope node/control plane uniquement |
| Kubescape | NSA sur `bad-practices` | 12 Failed | 50% | **Oui** — privileged, hostPID, hostNetwork, capabilities, non-root, no limits |
| Kubescape | NSA sur `client-alpha` | 5 Failed | 69% | — namespace propre, reste mineurs |
| Kubescape | Scan offline bad-pods.yaml | 5 findings | — | **Oui** — sans cluster, shift-left |
| Kubescape | Opérateur (continuous scan) | résultats par namespace | — | Scans auto, résultats en CRDs K8s natifs |

### Opérateur Kubescape (Helm)

5 pods dans le namespace `kubescape` :
- `kubescape` — scanner principal
- `kubevuln` — scan de vulnérabilités images
- `node-agent` — DaemonSet, observation runtime
- `operator` — orchestration
- `storage` — stockage résultats

Résultats consultables directement en `kubectl` :
- `kubectl get workloadconfigurationscansummaries -A` → vue globale par namespace
- `kubectl get workloadconfigurationscans -n <namespace>` → détail par workload

---

## Ce que fait chaque outil

| | kube-bench | Kubescape |
|---|---|---|
| **Scope** | Config du node/control plane | Node + workloads + RBAC + réseau |
| **Ce qu'il check** | Fichiers kubelet, API server, etcd | Pods, Deployments, ServiceAccounts, NetworkPolicies |
| **Benchmarks** | CIS uniquement | CIS + NSA/CISA + MITRE ATT&CK |
| **Comment il tourne** | Binaire sur le node (sudo) | CLI via kubeconfig (pas besoin de sudo) |
| **Scope namespace** | Non (cluster-level) | Oui (--include-namespaces) |
| **Scan YAML offline** | Non | Oui |

---

## Comparaison détaillée

### Communauté
- **kube-bench** : ~7k stars, maintenu par Aqua Security, releases régulières mais outil mature (moins de changements)
- **Kubescape** : ~10k stars, CNCF project (sandbox puis incubating), très actif, releases fréquentes

### Déploiement
- **kube-bench** : binaire à poser sur le node, ou Job K8s. Simple.
- **Kubescape** : `curl | bash` ou Helm. Encore plus simple.

### Support k3s
- **kube-bench** : supporte k3s via le benchmark `k3s-cis-1.24` mais il faut :
  1. Installer le dossier `cfg/` complet (sinon erreur "missing version_mapping")
  2. Passer `--config-dir /chemin/vers/cfg` et `--benchmark k3s-cis-1.24`
  3. Les checks etcd FAIL souvent car k3s utilise embedded etcd (4 FAIL sur 6 dans notre test)
  4. Certains audits plantent car `journalctl` ne trouve pas les logs k3s au path attendu
- **Kubescape** : fonctionne directement, pas de config spécifique. Frameworks dispo : `cis-v1.10.0`, `NSA`, `MITRE`, `SOC2`, etc.

### Rapports
- **kube-bench** : texte, JSON. C'est tout.
- **Kubescape** : JSON, PDF, SARIF, pretty-print, HTML. Intégration SARIF = direct dans GitHub/GitLab.

### Remédiation
- **kube-bench** : indique ce qui fail avec l'explication CIS, mais pas de fix suggéré
- **Kubescape** : propose des correctifs concrets pour chaque finding

### CI/CD
- **kube-bench** : pas d'intégration native, faut bricoler
- **Kubescape** : GitHub Action officielle, GitLab CI, Jenkins

### Licensing
- **kube-bench** : 100% open source (Apache 2.0)
- **Kubescape** : open source (Apache 2.0), backend cloud ARMO optionnel (dashboard, historique)

---

## Réponses aux questions techniques

### 1. Accès à l'API Kubernetes ?
- **kube-bench** : Non. Il lit les fichiers du filesystem du node directement. Il a besoin d'un accès root au node, pas à l'API.
- **Kubescape** : Oui. Il passe par l'API K8s pour lister les ressources.

### 2. Authentification
- **kube-bench** : Pas d'auth K8s. Il tourne sur le node avec sudo.
- **Kubescape** :
  - **CLI** : via kubeconfig (contexte courant), zéro config
  - **Opérateur** : via ServiceAccounts dédiés créés automatiquement par le Helm chart. Constaté dans le namespace `kubescape` :
    - `kubescape` (scanner principal)
    - `kubevuln` (scan de vulnérabilités images)
    - `node-agent` (observation runtime, DaemonSet)
    - `operator` (orchestration des scans)
    - `storage` (stockage des résultats en CRD)

### 3. RBAC nécessaire
- **kube-bench** : Aucun RBAC K8s. Juste sudo sur le node.
- **Kubescape** : un ClusterRole `kubescape` créé automatiquement par le Helm chart. Extrait réel du cluster :
  - `get`, `watch`, `list` sur : pods, namespaces, nodes, configmaps, services, serviceaccounts, secrets, deployments, statefulsets, daemonsets, replicasets, jobs, cronjobs, networkpolicies, ingresses, roles, rolebindings, clusterroles, clusterrolebindings, podsecuritypolicies, limitranges, resourcequotas, events, persistentvolumes, persistentvolumeclaims
  - `create`, `get`, `update`, `patch` sur les CRDs Kubescape : `workloadconfigurationscans`, `workloadconfigurationscansummaries`
  - `update` sur `namespaces` (pour poser des labels)
  - **Pas de `delete`, pas de `create` sur les ressources applicatives** → read-only sur le cluster, safe pour la prod

### 4. Scope namespace
- **kube-bench** : Non. Il check la config du node, pas les workloads. La notion de namespace n'a pas de sens.
- **Kubescape** :
  - **CLI** : `--include-namespaces client-alpha` → scan ciblé sur un namespace
  - **Opérateur** : les résultats sont stockés **par namespace** dans des CRDs. Constaté :
    - `kubectl get workloadconfigurationscans -n bad-practices` → 3 ressources scannées
    - `kubectl get workloadconfigurationscans -n client-alpha` → 4 ressources scannées

---

## Recommandation

Pour notre contexte :
- **Multi-tenant** (besoin de scanner par namespace) → Kubescape
- **k3s** (pas de complication de paths) → Kubescape
- **Besoin de scanner les workloads**  → Kubescape
- **CI/CD** (intégration future) → Kubescape
- **Shift-left** (scan YAML avant deploy) → Kubescape

**kube-bench** reste utile pour valider le hardening du node/control plane au moment de l'installation initiale du cluster. On pourrait l'utiliser en one-shot au setup puis Kubescape en continu.

### Proposition

1. **Kubescape** comme outil principal de validation continue
2. **kube-bench** en one-shot pour le hardening initial du cluster
3. Intégrer Kubescape dans le pipeline CI/CD (scan des manifests avant merge)

---

## Plan POC

1. Installer k3s sur une VM de test
2. Déployer des workloads représentatifs (ce qu'on a fait dans la demo)
3. Scanner avec les deux outils
4. Valider que Kubescape détecte bien les non-conformités par namespace
5. Tester l'intégration CI/CD (GitHub Action)
6. Valider le format des rapports (JSON pour exploitation, PDF pour la compliance)
7. Présenter les résultats à l'équipe sécu
