# CIS Kubernetes Benchmark : auditer la sécurité du cluster kubernetes

Le CIS Kubernetes Benchmark est un ensemble de recommandations de sécurité développé par le Center for Internet Security (CIS). kube-bench est l'outil open-source de référence pour vérifier automatiquement la conformité de votre cluster à ces guidelines. En quelques minutes, vous obtenez un rapport détaillé des points à corriger.

> Le CIS Benchmark fait partie du domaine Cluster Setup (15%) de la certification CKS. on doit savoir utiliser kube-bench et interpréter ses résultats.

---

## Qu'est-ce que le CIS Kubernetes Benchmark ?

Le CIS (Center for Internet Security) est une organisation à but non lucratif qui publie des guidelines de sécurité pour de nombreuses technologies. Le CIS Kubernetes Benchmark est la référence mondiale pour sécuriser un cluster Kubernetes.

### Ce que couvre le benchmark

Le benchmark analyse la configuration de tous les composants Kubernetes :

| Composant | Éléments vérifiés | Exemples |
|-----------|-------------------|----------|
| Control Plane | API Server, Controller Manager, Scheduler, etcd | Permissions des fichiers, arguments de sécurité |
| Worker Nodes | Kubelet, configuration réseau | Authentification kubelet, rotation des certificats |
| Policies | RBAC, Service Accounts, Network Policies | Principe du moindre privilège, segmentation réseau |
| Pod Security | Pod Security Standards, Security Contexts | Conteneurs non-root, capabilities |

### Les niveaux de recommandation

Chaque contrôle du benchmark est classé selon deux axes :

| Niveau | Description |
|--------|-------------|
| Level 1 | Recommandations de base, faciles à implémenter, impact minimal sur la fonctionnalité |
| Level 2 | Recommandations avancées, peuvent nécessiter plus de planification |
| Scored | Le contrôle affecte le score de conformité |
| Not Scored | Recommandation, mais pas comptabilisée dans le score |

La version 1.12 du CIS Kubernetes Benchmark couvre Kubernetes 1.32 à 1.34. kube-bench sélectionne généralement le profil adapté automatiquement, mais vérifiez la [matrice de compatibilité](https://github.com/aquasecurity/kube-bench/blob/main/docs/platforms.md).

---

## Ce que kube-bench ne fait pas

kube-bench vérifie uniquement la conformité CIS des configurations Kubernetes. Il ne couvre pas d'autres aspects de la sécurité :

| Ce que kube-bench fait ✅ | Ce que kube-bench ne fait pas ❌ |
|---------------------------|----------------------------------|
| Vérifier les permissions des fichiers de config | Scanner les vulnérabilités des images de conteneurs |
| Contrôler les arguments des composants K8s | Détecter les comportements suspects en runtime |
| Auditer les configurations RBAC | Analyser le trafic réseau |
| Vérifier les paramètres kubelet | Scanner les CVE des dépendances applicatives |

Pour une sécurité complète, combinez kube-bench avec :

- **Trivy** : scan des vulnérabilités d'images et configurations IaC
- **Falco** : détection runtime des comportements anormaux
- **Audit Logs** : traçabilité des actions sur l'API

---

## Installer kube-bench

kube-bench est développé par Aqua Security et disponible en open-source.

### Option 1 : Installation binaire (control plane)

Cette méthode est idéale pour les clusters autogérés où on a accès SSH aux nœuds :

```bash
# Créer le répertoire d'installation
sudo mkdir -p /opt/kube-bench

# Télécharger la dernière version (vérifiez sur GitHub)
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.15.0/kube-bench_0.15.0_linux_amd64.tar.gz -o /tmp/kube-bench.tar.gz

# Extraire les fichiers
tar -xvf /tmp/kube-bench.tar.gz -C /opt/kube-bench

# Ajouter au PATH
sudo mv /opt/kube-bench/kube-bench /usr/local/bin/
```

> ⚠️ Si on installe kube-bench via les archives GitHub, on conserve également le répertoire `cfg` extrait de l'archive. Il contient les profils de tests CIS indispensables à l'exécution. Le binaire seul ne suffit pas.

Vérification :

```bash
kube-bench version
# Output: 0.15.0
```

### Option 2 : Installation via package manager

Pour Debian/Ubuntu :

```bash
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.15.0/kube-bench_0.15.0_linux_amd64.deb -o kube-bench.deb
sudo dpkg -i kube-bench.deb
```

Pour RHEL/CentOS :

```bash
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.15.0/kube-bench_0.15.0_linux_amd64.rpm -o kube-bench.rpm
sudo rpm -ivh kube-bench.rpm
```

### Option 3 : Exécution en tant que Job Kubernetes

Cette méthode est recommandée pour les clusters managés (EKS, AKS, GKE) ou lorsqu'on a pas accès SSH aux nœuds :

```bash
# Déployer le job kube-bench (workers uniquement)
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Attendre la fin de l'exécution
kubectl wait --for=condition=complete job/kube-bench --timeout=60s

# Récupérer les résultats
kubectl logs job/kube-bench
```

Pour scanner le control plane et etcd depuis un Job (clusters autogérés), utilisez un Job avec `hostPID: true` et les volumes nécessaires :

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench-cp
  namespace: kube-system
spec:
  template:
    spec:
      hostPID: true
      nodeSelector:
        node-role.kubernetes.io/control-plane: ""
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      containers:
        - name: kube-bench
          image: aquasec/kube-bench:v0.15.0
          command: ["kube-bench", "run", "--targets", "controlplane,etcd,node"]
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
      restartPolicy: Never
      volumes:
        - name: var-lib-etcd
          hostPath:
            path: /var/lib/etcd
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
```

> L'option `hostPID: true` est nécessaire pour que kube-bench puisse détecter les processus `kube-apiserver`, `etcd`, etc. sur le nœud hôte.

---

## Exécuter les vérifications CIS

### Scan complet du cluster

Sur un nœud control plane (avec binaire installé), lancez un scan complet :

```bash
# Scan avec auto-détection de la version K8s et des fichiers de config
sudo kube-bench run
```

Le résultat affiche trois sections pour chaque contrôle :

```
[INFO] 1 Control Plane Security Configuration
[INFO] 1.1 Control Plane Node Configuration Files
[PASS] 1.1.1 Ensure that the API server pod specification file permissions are set to 600 or more restrictive (Automated)
[PASS] 1.1.2 Ensure that the API server pod specification file ownership is set to root:root (Automated)
[FAIL] 1.1.9 Ensure that the Container Network Interface file permissions are set to 600 or more restrictive (Manual)
[WARN] 1.1.12 Ensure that the etcd data directory ownership is set to etcd:etcd (Manual)
```

### Comprendre les résultats

| Statut | Signification | Action |
|--------|---------------|--------|
| PASS | Le contrôle est conforme | Aucune action requise |
| FAIL | Le contrôle échoue | Correction requise |
| WARN | Vérification manuelle recommandée | À investiguer |
| INFO | Information contextuelle | Lecture uniquement |

### Exporter les résultats

Pour archiver ou analyser les résultats :

```bash
# Export en fichier texte
sudo kube-bench run > kube-bench-report.txt

# Export en JSON (pour intégration CI/CD)
sudo kube-bench run --json > kube-bench-report.json

# Export en JUnit (pour intégration CI)
sudo kube-bench run --junit > kube-bench-report.xml
```

### Scanner des composants spécifiques

On peut cibler des composants particuliers :

```bash
# Scanner uniquement le control plane (API Server, Controller Manager, Scheduler)
sudo kube-bench run --targets controlplane

# Scanner uniquement les worker nodes
sudo kube-bench run --targets node

# Scanner uniquement etcd
sudo kube-bench run --targets etcd

# Scanner uniquement les policies
sudo kube-bench run --targets policies
```

---

## Interpréter et remédier aux problèmes

### Exemple de rapport

Voici un extrait typique avec les remédiations suggérées :

```
== Summary controlplane ==
41 checks PASS
9 checks FAIL
11 checks WARN
0 checks INFO

== Remediations controlplane ==
1.1.9 Run the below command (based on the file location on your system) on the control plane node.
For example, chmod 600 <path/to/cni/files>

1.1.12 On the etcd server node, get the etcd data directory, passed as an argument --data-dir,
from the command 'ps -ef | grep etcd'.
Run the below command (based on the etcd data directory found above).
For example, chown etcd:etcd /var/lib/etcd
```

### Corrections courantes

**1. Permissions des fichiers de configuration**

```bash
# Corriger les permissions de l'API Server
sudo chmod 600 /etc/kubernetes/manifests/kube-apiserver.yaml

# Corriger les permissions du scheduler
sudo chmod 600 /etc/kubernetes/manifests/kube-scheduler.yaml

# Corriger les permissions du controller manager
sudo chmod 600 /etc/kubernetes/manifests/kube-controller-manager.yaml
```

**2. Ownership des fichiers etcd**

```bash
# Vérifier le répertoire de données etcd
ps -ef | grep etcd | grep data-dir

# Corriger le propriétaire
sudo chown -R etcd:etcd /var/lib/etcd
```

**3. Arguments de sécurité de l'API Server**

Éditez `/etc/kubernetes/manifests/kube-apiserver.yaml` :

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    # Activer l'audit logging
    - --audit-log-path=/var/log/kubernetes/audit/audit.log
    - --audit-policy-file=/etc/kubernetes/audit-policy.yaml
    - --audit-log-maxage=30
    - --audit-log-maxbackup=10
    - --audit-log-maxsize=100
    # Désactiver les tokens anonymes
    - --anonymous-auth=false
    # Activer le chiffrement des secrets
    - --encryption-provider-config=/etc/kubernetes/enc/enc.yaml
```

**4. Configuration du kubelet**

Éditez `/var/lib/kubelet/config.yaml` :

```yaml
# Désactiver l'authentification anonyme
authentication:
  anonymous:
    enabled: false

# Activer la rotation des certificats
rotateCertificates: true

# Activer l'autorisation webhook
authorization:
  mode: Webhook
```

> Après modification des fichiers de configuration du control plane, l'API Server redémarre automatiquement (static pods). Pour les paramètres kubelet, redémarrez le service : `sudo systemctl restart kubelet`.

---

## Intégration CI/CD

### Pipeline GitLab CI

Ce pipeline exécute kube-bench à chaque merge request ou déploiement, génère des rapports JSON et JUnit, et échoue si des contrôles CIS sont en échec :

```yaml
security-audit:
  stage: security
  image: aquasec/kube-bench:v0.15.0
  script:
    # Générer le rapport JSON pour analyse
    - kube-bench run --json > kube-bench-results.json
    # Générer le rapport JUnit pour GitLab
    - kube-bench run --junit > kube-bench-results.xml
    - |
      FAIL_COUNT=$(jq '[.Controls[].results[] | select(.status == "FAIL")] | length' kube-bench-results.json)
      if [ "$FAIL_COUNT" -gt "0" ]; then
        echo "❌ $FAIL_COUNT contrôles CIS en échec"
        exit 1
      fi
  artifacts:
    paths:
      - kube-bench-results.json
    reports:
      junit: kube-bench-results.xml
```

### CronJob Kubernetes

Pour des audits réguliers :

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: kube-bench-scheduled
  namespace: security
spec:
  schedule: "0 2 * * *"  # Tous les jours à 2h
  jobTemplate:
    spec:
      template:
        spec:
          hostPID: true
          containers:
          - name: kube-bench
            image: aquasec/kube-bench:latest
            command: ["kube-bench"]
            args: ["run", "--json"]
            volumeMounts:
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
          volumes:
          - name: var-lib-kubelet
            hostPath:
              path: /var/lib/kubelet
          - name: etc-kubernetes
            hostPath:
              path: /etc/kubernetes
          restartPolicy: Never
```

---

## Faux positifs et distributions spécifiques

kube-bench peut signaler des échecs qui sont en réalité des faux positifs, selon la distribution Kubernetes utilisée.

### Distributions avec profils dédiés

kube-bench détecte automatiquement certaines distributions et applique le profil approprié :

| Distribution | Profil | Notes |
|--------------|--------|-------|
| RKE2 | rke2-cis-* | Fichiers dans /etc/rancher/rke2/, etcd intégré |
| k3s | k3s-cis-* | Architecture légère, chemins non standards |
| OpenShift | ocp-4.* | Profil Red Hat spécifique |
| kubeadm | cis-* | Profil CIS standard |
| microk8s | Non supporté nativement | Chemins snap non détectés |

### Exemples de faux positifs courants

> Ne corrigez pas aveuglément tous les `FAIL`. Certains peuvent être intentionnels ou liés à votre distribution.

**RKE2/k3s :**
- Les fichiers de configuration sont dans `/etc/rancher/` au lieu de `/etc/kubernetes/`
- etcd peut être intégré au binaire, pas en tant que processus séparé

**Kubespray :**
- Le propriétaire etcd peut être `root` au lieu de `etcd:etcd` selon la configuration
- Certains arguments de sécurité sont dans des fichiers de drop-in systemd

**Clusters managés :**
- Les checks du control plane échouent car on n'a pas accès aux nœuds masters

### Documenter les exceptions

Créez un fichier d'exceptions pour votre équipe :

```yaml
# exceptions-cis.yaml - Contrôles ignorés volontairement
exceptions:
  - control: "1.1.12"
    reason: "etcd géré par RKE2, ownership root attendu"
    reviewed_by: "security-team"
    date: "2026-03-15"
  - control: "4.2.6"
    reason: "ProtectKernelDefaults désactivé pour compatibilité drivers GPU"
    reviewed_by: "security-team"
    date: "2026-03-15"
```

---

## Alternatives à kube-bench

| Outil | Avantages | Limites |
|-------|-----------|---------|
| kube-bench | Référence, simple, CNCF | Uniquement CIS Benchmark |
| Kubescape | Multi-frameworks (CIS, NSA, MITRE) | Plus complexe |
| Checkov | Multi-cloud, IaC | Focus IaC, moins K8s runtime |
| Trivy | Scanner polyvalent | CIS = une partie des features |

---

## Bonnes pratiques

1. **Planifier les audits** : Exécutez kube-bench au minimum mensuellement, idéalement à chaque changement de configuration
2. **Prioriser les corrections** : Commencez par les contrôles `FAIL` de Level 1
3. **Documenter les exceptions** : Certains contrôles peuvent être inapplicables à votre contexte
4. **Automatiser** : Intégrez kube-bench dans votre CI/CD
5. **Suivre les versions** : Mettez à jour kube-bench avec votre version Kubernetes

---

## À retenir

- Le CIS Kubernetes Benchmark définit les standards de sécurité pour Kubernetes
- kube-bench automatise la vérification de conformité
- Les résultats indiquent PASS, FAIL, WARN pour chaque contrôle
- Les remédiations sont fournies directement dans le rapport
- Sur les clusters managés, seuls les worker nodes et policies sont vérifiables
- L'intégration CI/CD permet des audits automatiques et continus

---

## Prochaines étapes

- [Falco : détection runtime](https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/kubernetes/securiser/falco/)
- [Audit Logs Kubernetes](https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/kubernetes/securiser/audit-logs/)
- [Network Policies](https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/kubernetes/network-policies/)
- [Pod Security Standards](https://blog.stephane-robert.info/docs/conteneurs/orchestrateurs/kubernetes/securiser/pod-security-standards/)
