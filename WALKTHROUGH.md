# Walkthrough Détaillé - Déploiement Kubernetes sur Hetzner

Ce guide détaille l'ensemble du processus de mise en place d'un cluster Kubernetes sur Hetzner avec ArgoCD et une application de chat.

## Table des Matières

1. [Préparation de l'Infrastructure](#1-préparation-de-linfrastructure)
2. [Configuration du Cluster Kubernetes](#2-configuration-du-cluster-kubernetes)
3. [Installation de Calico (CNI)](#3-installation-de-calico-cni)
4. [Configuration du Cloud Controller Manager](#4-configuration-du-cloud-controller-manager)
5. [Installation du CSI Driver](#5-installation-du-csi-driver)
6. [Déploiement de Traefik](#6-déploiement-de-traefik)
7. [Configuration du Stockage](#7-configuration-du-stockage)
8. [Installation d'ArgoCD](#8-installation-dargocd)
9. [Déploiement de l'Application](#9-déploiement-de-lapplication)
10. [Configuration de Cert-Manager](#10-configuration-de-cert-manager)
11. [Mise en Place du CI/CD](#11-mise-en-place-du-cicd)
12. [Vérification et Tests](#12-vérification-et-tests)

---

## 1. Préparation de l'Infrastructure

### 1.1 Achat des VPS Hetzner

1. Se connecter à [Hetzner Cloud Console](https://console.hetzner.cloud/)
2. Créer un nouveau projet
3. Acheter 2 VPS dans la région **Helsinki (hel1)**
   - Configuration recommandée: 4GB RAM minimum par nœud
   - OS: Ubuntu 22.04 LTS

### 1.2 Génération du Token API

1. Aller dans **Security** > **API Tokens**
2. Créer un nouveau token avec permissions **Read & Write**
3. Sauvegarder le token en lieu sûr

### 1.3 Configuration SSH

```bash
# Se connecter au master node
ssh root@<IP_MASTER>

# Configurer les clés SSH entre les nœuds si nécessaire
```

---

## 2. Configuration du Cluster Kubernetes

### 2.1 Installation de Kubernetes

Sur chaque nœud, installer les composants nécessaires:

```bash
# Mise à jour du système
apt-get update && apt-get upgrade -y

# Installation de Docker/containerd
apt-get install -y containerd

# Installation de kubeadm, kubelet, kubectl
# (suivre la documentation officielle Kubernetes)
```

### 2.2 Initialisation du Cluster

Sur le master node:

```bash
kubeadm init --pod-network-cidr=192.168.0.0/16

# Configurer kubectl pour l'utilisateur
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```

### 2.3 Rejoindre les Worker Nodes

Sur chaque worker node:

```bash
# Utiliser la commande fournie par kubeadm init
kubeadm join <IP_MASTER>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

---

## 3. Installation de Calico (CNI)

Calico est le plugin réseau pour permettre la communication entre les pods.

```bash
# Installation de Calico
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml

# Vérifier l'installation
kubectl get pods -n kube-system | grep calico
```

---

## 4. Configuration du Cloud Controller Manager

### 4.1 Le Problème

Par défaut, Kubernetes ne sait pas interagir avec l'API Hetzner. Le service LoadBalancer reste en statut `Pending`.

### 4.2 Création du Secret

Créer le secret avec le token API Hetzner:

```bash
kubectl create secret generic hcloud \
  --from-literal=token=<VOTRE_HETZNER_API_TOKEN> \
  -n kube-system
```

### 4.3 Installation du CCM

```bash
kubectl apply -f https://raw.githubusercontent.com/hetznercloud/hcloud-cloud-controller-manager/master/deploy/ccm.yaml
```

### 4.4 Vérification

```bash
# Vérifier que le CCM est en cours d'exécution
kubectl get pods -n kube-system | grep hcloud

# Les pods doivent être en statut Running
```

---

## 5. Installation du CSI Driver

Le CSI Driver permet de provisionner dynamiquement des volumes persistants depuis Hetzner.

### 5.1 Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/hetznercloud/csi-driver/refs/heads/main/deploy/kubernetes/hcloud-csi.yml
```

### 5.2 Vérification

```bash
# Vérifier les pods du CSI driver
kubectl get pods -n kube-system | grep csi

# Doit afficher:
# - hcloud-csi-controller-0
# - hcloud-csi-node-xxxxx (un par nœud)
```

---

## 6. Déploiement de Traefik

Traefik est l'Ingress Controller qui gère le routage HTTP/HTTPS.

### 6.1 Problème Initial

Sans spécifier la région, le LoadBalancer ne se provisionne pas.

### 6.2 Installation Correcte

```bash
# Ajouter le repo Helm
helm repo add traefik https://traefik.github.io/charts
helm repo update

# Installation avec la région spécifiée
helm upgrade --install traefik traefik/traefik \
  --namespace traefik --create-namespace \
  --set service.type=LoadBalancer \
  --set service.annotations."load-balancer\.hetzner\.cloud/location"=hel1
```

### 6.3 Récupération de l'IP Publique

```bash
# Attendre que le LoadBalancer soit provisionné
kubectl get svc -n traefik traefik -w

# Récupérer l'IP externe
kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

---

## 7. Configuration du Stockage

### 7.1 Création de la StorageClass

Créer le fichier `apps/infrastructure/templates/storage-class.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: hcloud-volumes
provisioner: csi.hetzner.cloud
parameters:
  type: hdd  # ou ssd pour de meilleures performances
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 7.2 Explication

- **provisioner**: Utilise le CSI driver Hetzner
- **type**: `hdd` (moins cher) ou `ssd` (plus rapide)
- **reclaimPolicy**: `Delete` supprime le volume quand le PVC est supprimé
- **volumeBindingMode**: `WaitForFirstConsumer` attend qu'un pod utilise le PVC avant de créer le volume

---

## 8. Installation d'ArgoCD

ArgoCD implémente le pattern GitOps pour le déploiement continu.

### 8.1 Installation

```bash
# Créer le namespace
kubectl create namespace argocd

# Installer ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 8.2 Accès à l'Interface

```bash
# Récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Port-forward pour accéder localement
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Accéder à https://localhost:8080
# Username: admin
# Password: (celui récupéré ci-dessus)
```

### 8.3 Configuration du Repository

Dans l'interface ArgoCD:

1. **Settings** > **Repositories** > **Connect Repo**
2. URL: `https://github.com/Sipixer/ynov-k8s-orchestration`
3. Type: Public (ou configurer les credentials)

---

## 9. Déploiement de l'Application

### 9.1 Structure Helm Chart

L'application est structurée en Helm Chart:

```
apps/chat-app/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── namespace.yaml
    ├── redis.yaml
    ├── chat-server.yaml
    └── ingress.yaml
```

### 9.2 Déploiement avec ArgoCD

Appliquer l'Application root:

```bash
kubectl apply -f https://raw.githubusercontent.com/Sipixer/ynov-k8s-orchestration/refs/heads/main/bootstrap/root-app.yaml
```

Cette application ArgoCD va déployer:
- Infrastructure (StorageClass)
- Cert-Manager
- Chat-App (Redis + Chat Server)

### 9.3 Comportement du Redis

Le pod Redis va probablement redémarrer plusieurs fois au début:

1. Le pod démarre sans volume
2. Le PVC demande un volume
3. Le CSI driver provisionne le volume chez Hetzner
4. Le volume est attaché au nœud
5. Le pod redémarre et monte le volume
6. Le pod devient stable

C'est normal, patience.

### 9.4 Vérification

```bash
# Vérifier les pods
kubectl get pods -n chat-app

# Vérifier les PVC
kubectl get pvc -n chat-app

# Vérifier les volumes Hetzner
kubectl get pv
```

---

## 10. Configuration de Cert-Manager

Cert-Manager automatise la création et le renouvellement des certificats SSL/TLS.

### 10.1 Installation

Déjà inclus dans la configuration ArgoCD, mais manuellement:

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

### 10.2 Configuration du ClusterIssuer

Le fichier `apps/cert-manager/templates/cluster-issuer.yaml` configure Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: votre-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

### 10.3 Vérification

```bash
# Vérifier le ClusterIssuer
kubectl get clusterissuer

# Vérifier les certificats
kubectl get certificate -n chat-app
```

---

## 11. Mise en Place du CI/CD

### 11.1 Repository Application

Le repo [server-chat](https://github.com/Sipixer/server-chat) contient:
- Le code source de l'application
- Le Dockerfile
- La GitHub Action pour le build

### 11.2 Workflow GitHub Actions

Le workflow `.github/workflows/build.yml` fait:

1. Build de l'image Docker
2. Tag avec le SHA du commit
3. Push sur ghcr.io
4. Clone du repo d'orchestration
5. Mise à jour de `apps/chat-app/values.yaml` avec le nouveau tag
6. Commit et push du changement

### 11.3 Déploiement Automatique

1. ArgoCD poll le repo toutes les 3 minutes
2. Détecte le changement dans `values.yaml`
3. Applique la nouvelle configuration
4. Kubernetes effectue un rolling update

### 11.4 Configuration du Token GitHub

Pour que le workflow puisse push sur le repo d'orchestration:

1. Créer un Personal Access Token (PAT) avec scope `repo`
2. L'ajouter comme secret dans le repo application: `ORCHESTRATION_TOKEN`

---

## 12. Vérification et Tests

### 12.1 DNS

Configurer le domaine DuckDNS pour pointer vers l'IP du LoadBalancer:

```bash
# Récupérer l'IP
kubectl get svc -n traefik traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Configurer sur DuckDNS
curl "https://www.duckdns.org/update?domains=sylvain-chat&token=<DUCKDNS_TOKEN>&ip=<IP>"
```

### 12.2 Test SSL

```bash
# Vérifier que le certificat est bien généré
curl https://sylvain-chat.duckdns.org

# Ou ouvrir dans le navigateur
```

### 12.3 Test de l'Application

1. Ouvrir `https://sylvain-chat.duckdns.org`
2. Créer un compte
3. Envoyer un message
4. Vérifier la persistance (redémarrer le pod chat-server, les messages doivent rester)

### 12.4 Test du CI/CD

1. Faire un changement dans le code de l'application
2. Commit et push sur `main`
3. Attendre le build GitHub Actions
4. Vérifier qu'ArgoCD détecte le changement
5. Vérifier le rolling update dans le cluster

```bash
# Suivre le déploiement
kubectl rollout status deployment/chat-server -n chat-app

# Vérifier l'historique
kubectl rollout history deployment/chat-server -n chat-app
```

---

## Troubleshooting Avancé

### Problème: LoadBalancer reste en Pending

```bash
# Vérifier les logs du CCM
kubectl logs -n kube-system -l app=hcloud-cloud-controller-manager

# Vérifier que le secret existe
kubectl get secret hcloud -n kube-system

# Vérifier la configuration de la région
kubectl get svc -n traefik traefik -o yaml | grep location
```

### Problème: Volume ne se provisionne pas

```bash
# Vérifier les logs du CSI controller
kubectl logs -n kube-system hcloud-csi-controller-0 -c hcloud-csi-driver

# Vérifier les événements
kubectl describe pvc redis-pvc -n chat-app

# Vérifier la StorageClass
kubectl get storageclass
```

### Problème: ArgoCD ne synchronise pas

```bash
# Vérifier les logs d'ArgoCD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller

# Forcer la synchronisation
kubectl patch application chat-app -n argocd --type merge -p '{"operation":{"initiatedBy":{"username":"admin"},"sync":{"syncStrategy":{"hook":{}}}}}'
```

### Problème: Certificat SSL non généré

```bash
# Vérifier cert-manager
kubectl get pods -n cert-manager

# Vérifier les certificats
kubectl describe certificate -n chat-app

# Vérifier les challenges
kubectl get challenges -n chat-app
```

---

## Améliorations Futures

1. **Monitoring**: Installer Prometheus + Grafana
2. **Logging**: Mettre en place ELK stack ou Loki
3. **Backup**: Configurer Velero pour les backups
4. **Autoscaling**: Configurer HPA (Horizontal Pod Autoscaler)
5. **Multi-region**: Étendre le cluster sur plusieurs régions
6. **Security**: Mettre en place OPA/Gatekeeper pour les policies
7. **Service Mesh**: Considérer Istio ou Linkerd

---

## Ressources Utiles

- [Documentation Hetzner Cloud](https://docs.hetzner.com/cloud/)
- [Documentation Kubernetes](https://kubernetes.io/docs/)
- [Documentation ArgoCD](https://argo-cd.readthedocs.io/)
- [Documentation Traefik](https://doc.traefik.io/traefik/)
- [Documentation Cert-Manager](https://cert-manager.io/docs/)
- [Documentation Helm](https://helm.sh/docs/)

---

## Conclusion

Ce setup offre une infrastructure moderne et scalable avec:
- Déploiements automatisés via GitOps
- SSL/TLS automatique
- Stockage persistant
- Monitoring des déploiements
- Pipeline CI/CD complète

Le tout pour environ 10-15€/mois sur Hetzner.
