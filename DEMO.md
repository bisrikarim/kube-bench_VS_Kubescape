# POC Sécurité Cluster — kube-bench vs Kubescape

---

## Partie 0 — Clean complet avant de commencer

```bash
# désinstaller l'opérateur kubescape si présent
helm uninstall kubescape -n kubescape 2>/dev/null; kubectl delete ns kubescape --ignore-not-found

# supprimer les workloads de test
kubectl delete -f workloads/bad-pods.yaml --ignore-not-found
kubectl delete -f workloads/good-pod.yaml --ignore-not-found
kubectl delete -f workloads/namespaces.yaml --ignore-not-found

# supprimer kube-bench
sudo rm -f /usr/local/bin/kube-bench
sudo rm -rf /etc/kube-bench
rm -rf /tmp/cfg /tmp/kube-bench*

# supprimer kubescape
rm -rf ~/.kubescape

# désinstaller k3s complètement
/usr/local/bin/k3s-uninstall.sh 2>/dev/null

# vérifier qu'il ne reste rien
kubectl get pods -A 2>/dev/null || echo "Cluster supprimé ✓"
which kube-bench 2>/dev/null || echo "kube-bench supprimé ✓"
which kubescape 2>/dev/null || echo "kubescape supprimé ✓"
```

---

## Partie 1 — Monter le cluster k3s

```bash
# installer k3s
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --write-kubeconfig-mode 644 --disable traefik" sh -

# configurer kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
export KUBECONFIG=~/.kube/config

# vérifier que ça tourne
kubectl get nodes
kubectl get pods -A
```

---

## Partie 2 — Déployer des workloads de test

On va créer des namespaces, et à côté des pods volontairement "mal configurés" pour voir si les outils les détectent.

### Ce qu'on déploie

**Namespace `client-alpha`** — simule un client AWX bien configuré :
- `awx-web` (nginx) : non-root, readOnlyRootFilesystem, drop ALL capabilities, resource limits, probes, seccompProfile RuntimeDefault
- `awx-postgres` (postgres:15) : non-root, drop ALL, resource limits, probes, fsGroup
- automountServiceAccountToken: false sur les deux

→ C'est ce qu'on devrait avoir en prod.

**Namespace `bad-practices`** — 6 pods volontairement pourris :
| Pod | Ce qui est mal |
|---|---|
| `bad-privileged` | `privileged: true` + `runAsUser: 0` (root) |
| `bad-no-limits` | Aucun resource request/limit |
| `bad-host-ns` | `hostNetwork: true` + `hostPID: true` |
| `bad-capabilities` | Capabilities `NET_ADMIN` + `SYS_ADMIN` ajoutées |
| `bad-hostpath` | Monte `/etc` du host en volume |
| `bad-everything` | Image `:latest`, pas de probes, pas de securityContext, rien |

→ Exactement le genre de trucs qu'un outil de sécu doit détecter.

```bash
# créer les namespaces
kubectl apply -f workloads/namespaces.yaml

# déployer un workload propre (simule AWX côté client)
kubectl apply -f workloads/good-pod.yaml

# déployer des workloads pourris (pour tester la détection)
kubectl apply -f workloads/bad-pods.yaml

# vérifier que tout tourne
kubectl get pods -A
```

---

## Partie 3 — Test avec kube-bench

kube-bench c'est un outil d'Aqua Security qui check les recommandations CIS Benchmark. Il vérifie la config du cluster lui-même (kubelet, API server, etcd, etc.).

### Installer

```bash
# télécharger l'archive complète (binaire + fichiers de config CIS)
curl -sL https://github.com/aquasecurity/kube-bench/releases/download/v0.8.0/kube-bench_0.8.0_linux_amd64.tar.gz -o /tmp/kube-bench.tar.gz
cd /tmp && tar -xzf kube-bench.tar.gz
sudo cp kube-bench /usr/local/bin/
sudo chmod +x /usr/local/bin/kube-bench

# important : les fichiers de config CIS sont dans le dossier cfg/
# kube-bench en a besoin sinon il plante ("missing version_mapping")
```

### Scanner

```bash
# scan avec le benchmark k3s (il faut sudo car il lit les fichiers du node)
# --config-dir pointe vers le dossier cfg/ extrait de l'archive
sudo kube-bench run --benchmark k3s-cis-1.24 --config-dir /tmp/cfg

# voir juste les FAIL
sudo kube-bench run --benchmark k3s-cis-1.24 --config-dir /tmp/cfg | grep "\[FAIL\]"

# résultat en JSON
sudo kube-bench run --benchmark k3s-cis-1.24 --config-dir /tmp/cfg --json > results/kube-bench.json
```

### Ce qu'on observe

Résultat : **42 PASS, 7 FAIL, 49 WARN, 27 INFO**

- Les FAIL sont sur etcd (normal : k3s utilise embedded etcd, pas le même setup que kubeadm)
- kube-bench check la config du **node** et du **control plane**
- Il ne regarde **PAS les workloads** (pods, deployments)
- Il a besoin d'un accès **root** au node (sudo), pas juste l'API K8s
- Il ne voit pas nos pods "bad-practices" → c'est **normal**, c'est pas son scope
- Note : sur k3s il faut préciser `--benchmark k3s-cis-1.24` et `--config-dir`

---

## Partie 4 — Test avec Kubescape

Kubescape c'est un outil de ARMO qui check la sécurité à tous les niveaux : config cluster + workloads + RBAC + network policies. Il supporte CIS, NSA/CISA, et MITRE ATT&CK.

### Installer

```bash
curl -s https://raw.githubusercontent.com/kubescape/kubescape/master/install.sh | /bin/bash
export PATH=$PATH:$HOME/.kubescape/bin
```

### Scanner

```bash
# voir les frameworks disponibles
kubescape list frameworks

# scan CIS complet du cluster
kubescape scan framework cis-v1.10.0

# scan NSA/CISA
kubescape scan framework NSA

# scan MITRE ATT&CK
kubescape scan framework MITRE

# scan ciblé sur un namespace (comme on ferait pour un client)
kubescape scan framework NSA --include-namespaces client-alpha

# scan ciblé sur les bad practices
kubescape scan framework NSA --include-namespaces bad-practices

# scan sur un fichier YAML (sans cluster, offline — shift left)
kubescape scan workloads/bad-pods.yaml

# export JSON pour exploitation
kubescape scan framework cis-v1.10.0 --format json --output results/kubescape-cis.json

# export PDF
kubescape scan framework cis-v1.10.0 --format pdf --output results/kubescape-cis.pdf
```

### Ce qu'on observe

Résultats :
- **bad-practices** (NSA) : 12 contrôles failed, score 50% → tout détecté
- **client-alpha** (NSA) : 5 contrôles failed, score 69% → beaucoup plus propre
- **bad-pods.yaml** (offline) : 5 findings, top 3 "highest-stake workloads" identifiés

Points clés :
- Kubescape voit les workloads → il **détecte** nos pods mal configurés (privileged, hostPID, hostPath, capabilities, non-root...)
- Il peut scanner **par namespace** (`--include-namespaces`) → exactement ce qu'il nous faut pour nos clients
- Il peut scanner des YAML **avant même de déployer** (shift-left en CI/CD)
- Pas besoin de sudo, il passe par l'API Kubernetes via kubeconfig
- Il classe les workloads par niveau de risque ("highest-stake workloads")

### Rapports générés

| Commande | Format | Fichier | Usage |
|---|---|---|---|
| `--format json --output results/kubescape-cis.json` | JSON | `results/kubescape-cis.json` | Exploitation programmatique, intégration CI/CD, parsing jq |
| `--format pdf --output results/kubescape-cis.pdf` | PDF | `results/kubescape-cis.pdf` | Livrable pour l'équipe compliance / audit SOC |
| `--format sarif --output results/kubescape-cis.sarif` | SARIF | `results/kubescape-cis.sarif` | Intégration GitHub Security / GitLab SAST |
| (par défaut) | pretty-print | terminal | Lecture humaine pendant la démo |
| `sudo kube-bench run --json` | JSON | `results/kube-bench.json` | Seul format structuré de kube-bench |
| `sudo kube-bench run` | texte | terminal | Lecture humaine, c'est tout |

→ Kubescape : 4 formats de sortie. kube-bench : texte ou JSON, c'est tout.

---

## Partie 5 — Comparaison directe

Après les tests, on regarde les résultats côte à côte :

```bash
# kube-bench : 42 PASS, 7 FAIL, 49 WARN → que du node/control plane
sudo kube-bench run --benchmark k3s-cis-1.24 --config-dir /tmp/cfg 2>/dev/null | tail -5

# est-ce que kube-bench détecte nos bad pods ?
# → Non. Aucun finding sur privileged, hostPath, root containers. C'est pas son scope.

# kubescape sur bad-practices : 12 contrôles failed, score 50%
kubescape scan framework NSA --include-namespaces bad-practices
# → Détecte tout : privileged, hostPID, hostNetwork, capabilities, non-root, no limits...

# kubescape sur client-alpha : 5 contrôles failed, score 69%
kubescape scan framework NSA --include-namespaces client-alpha
# → Beaucoup plus propre, reste des mineurs (credentials en env, NetworkPolicy)

# scan offline sur le YAML (sans cluster)
kubescape scan workloads/bad-pods.yaml
# → Détecte les problèmes avant même de déployer
```

---

## Partie 6 — Opérateur Kubescape (mode continu)

Kubescape peut aussi tourner en continu dans le cluster comme opérateur.

```bash
# installer via Helm
helm repo add kubescape https://kubescape.github.io/helm-charts/
helm repo update
helm upgrade --install kubescape kubescape/kubescape-operator \
  -n kubescape --create-namespace \
  --set clusterName=$(kubectl config current-context) \
  --set capabilities.continuousScan=enable

# vérifier les pods
kubectl get pods -n kubescape

# voir les ServiceAccounts créés (→ répond à Q2 auth)
kubectl get serviceaccounts -n kubescape

# voir le ClusterRole et les droits RBAC (→ répond à Q3 RBAC)
kubectl get clusterrole kubescape -o yaml

# voir les résultats de compliance par namespace (→ répond à Q4 scope)
kubectl get workloadconfigurationscansummaries -A
kubectl get workloadconfigurationscans -n bad-practices
kubectl get workloadconfigurationscans -n client-alpha
```

Ce qu'on montre :
- L'opérateur crée ses propres ServiceAccounts (kubescape, kubevuln, node-agent, operator, storage)
- Le ClusterRole est en **lecture seule** sur les ressources du cluster (get/watch/list)
- Les résultats sont stockés **par namespace** dans des CRDs → cloisonnement natif
- Le scan tourne en continu, pas besoin de relancer manuellement

---

## Partie 7 — Nettoyage

```bash
# supprimer l'opérateur kubescape
helm uninstall kubescape -n kubescape
kubectl delete ns kubescape

# supprimer les workloads de test
kubectl delete -f workloads/bad-pods.yaml
kubectl delete -f workloads/good-pod.yaml
kubectl delete -f workloads/namespaces.yaml

# désinstaller k3s (si on veut)
/usr/local/bin/k3s-uninstall.sh
```

---

## Notes perso / Points à retenir pour la réunion

Voir `SYNTHESE.md`
