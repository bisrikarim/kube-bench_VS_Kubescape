# POC Sécurité Cluster — kube-bench vs Kubescape

> Demo interne — Validation sécurité du cluster k3s (contexte AWX multi-tenant)

---

## Prérequis

- WSL2 avec Ubuntu 22.04
- Accès sudo
- Connexion internet (pour les install)

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

On va créer un environnement qui ressemble à notre prod : des namespaces clients avec des pods AWX, et à côté des pods volontairement "mal configurés" pour voir si les outils les détectent.

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

## Partie 6 — Nettoyage

```bash
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
