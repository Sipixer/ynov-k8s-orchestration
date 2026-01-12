# Présentation - Déploiement GitOps sur Kubernetes

## Slide 1 - Introduction

**Projet: Application de Chat avec GitOps**

- Déploiement d'une application web en production
- Infrastructure as Code complète
- Pipeline CI/CD automatisé
- Hébergement sur Hetzner Cloud

---

## Slide 2 - Architecture Globale

**Stack Technologique**

- **Infrastructure**: Hetzner Cloud (2 VPS)
- **Orchestration**: Kubernetes avec Calico (CNI)
- **Ingress**: Traefik avec LoadBalancer
- **GitOps**: ArgoCD
- **CI/CD**: GitHub Actions
- **SSL/TLS**: Cert-Manager + Let's Encrypt
- **Stockage**: Hetzner CSI Driver

---

## Slide 3 - L'Application

**Chat Application**

**Backend:**
- Node.js/Express
- WebSocket pour le temps réel
- API REST

**Base de données:**
- Redis avec stockage persistant
- Volume HDD 1Gi provisionné dynamiquement

**Accès:**
- https://sylvain-chat.duckdns.org
- Certificat SSL automatique

---

## Slide 4 - Infrastructure Kubernetes

**Composants Installés**

1. **Calico**: Réseau CNI pour la communication inter-pods
2. **Hetzner Cloud Controller Manager**: Provisionnement automatique des LoadBalancers
3. **Hetzner CSI Driver**: Création dynamique de volumes persistants
4. **Traefik**: Ingress Controller pour le routage HTTP/HTTPS
5. **Cert-Manager**: Génération et renouvellement automatique des certificats SSL

**Spécificité Hetzner:** Annotation de région (`hel1`) obligatoire pour les LoadBalancers

---

## Slide 5 - GitOps avec ArgoCD

**Principe GitOps**

- Git comme source de vérité unique
- Déclaratif: on décrit l'état désiré, pas les actions
- Synchronisation automatique

**ArgoCD**

- Poll le repository toutes les 3 minutes
- Détecte les changements dans les manifests
- Applique automatiquement les modifications
- Interface web pour visualiser l'état du cluster

**Pattern App-of-Apps:** Une application ArgoCD qui déploie d'autres applications

---

## Slide 6 - Structure du Repository GitOps

```
ynov-k8s-orchestration/
├── apps/
│   ├── infrastructure/          # StorageClass
│   ├── cert-manager/           # Configuration SSL
│   └── chat-app/               # Application (Helm Chart)
│       ├── Chart.yaml
│       ├── values.yaml         # Configuration modifiable
│       └── templates/          # Manifests Kubernetes
├── bootstrap/
│   └── root-app.yaml           # ArgoCD App-of-Apps
└── .github/workflows/
    └── update-chat-app.yml     # CI/CD
```

**Helm Charts:** Templating des manifests Kubernetes avec variables

---

## Slide 7 - Pipeline CI/CD - Vue d'Ensemble

**Workflow Automatique en 5 Étapes**

1. Développeur push du code
2. Build et publication de l'image Docker
3. Notification au repository GitOps
4. Mise à jour automatique du tag d'image
5. Déploiement par ArgoCD

**Avantages:**
- Déploiement automatique en production
- Traçabilité complète via Git
- Rollback facile
- Zero-downtime avec rolling updates

---

## Slide 8 - Pipeline CI/CD - Détail Technique

**Repository Application (server-chat)**

GitHub Actions:
1. Build de l'image Docker
2. Tag avec le SHA du commit (`main-abc123`)
3. Push vers GitHub Container Registry (ghcr.io)
4. Envoie un `repository_dispatch` au repo GitOps

**Repository GitOps (ynov-k8s-orchestration)**

GitHub Actions:
1. Reçoit l'événement `update-chat-app-image`
2. Checkout du repository
3. Met à jour `values.yaml` avec le nouveau tag (via `sed`)
4. Commit et push du changement

**ArgoCD:**
- Détecte le changement dans `values.yaml`
- Pull la nouvelle image depuis ghcr.io
- Rolling update sur Kubernetes

---

## Slide 9 - Diagramme de Séquence CI/CD

```
Développeur → server-chat (push)
    ↓
GitHub Actions (App)
    ↓ build & push
ghcr.io (nouvelle image)
    ↓ repository_dispatch
ynov-k8s-orchestration
    ↓
GitHub Actions (Ops) → update values.yaml
    ↓ commit & push
ynov-k8s-orchestration (nouveau tag)
    ↓ poll (3min)
ArgoCD (sync détecté)
    ↓ pull image
ghcr.io
    ↓ rolling update
Cluster Kubernetes
    ↓
Application mise à jour !
```

---

## Slide 10 - Gestion du Stockage Persistant

**Problématique**

Redis doit persister les données entre les redémarrages

**Solution: Volumes Persistants**

1. **StorageClass**: Définit comment provisionner les volumes
   - Provisioner: `csi.hetzner.cloud`
   - Type: HDD (économique)
   - Binding: `WaitForFirstConsumer` (création à la demande)

2. **PersistentVolumeClaim (PVC)**: Demande de volume
   - 1Gi de stockage
   - AccessMode: ReadWriteOnce

3. **CSI Driver**: Communique avec l'API Hetzner
   - Crée le volume cloud
   - L'attache au nœud approprié

**Résultat:** Redis redémarre plusieurs fois au début (normal), puis se stabilise

---

## Slide 11 - Gestion SSL/TLS Automatique

**Cert-Manager + Let's Encrypt**

**Composants:**

1. **Cert-Manager**: Opérateur Kubernetes pour gérer les certificats
2. **ClusterIssuer**: Configuration Let's Encrypt
3. **Certificate**: Ressource automatiquement créée

**Processus Automatique:**

1. L'Ingress demande un certificat via annotation
2. Cert-Manager crée un Challenge HTTP-01
3. Let's Encrypt valide le domaine
4. Certificat généré et stocké en Secret
5. Traefik utilise le certificat pour le TLS
6. Renouvellement automatique avant expiration

**Résultat:** HTTPS automatique sans configuration manuelle

---

## Slide 12 - LoadBalancer sur Hetzner

**Problème Initial**

Kubernetes ne sait pas comment créer des LoadBalancers sur Hetzner

**Solution: Cloud Controller Manager (CCM)**

1. Déploiement du Hetzner CCM
2. Secret avec token API Hetzner
3. Annotation de région obligatoire (`hel1`)

**Fonctionnement:**

- Service type `LoadBalancer` créé
- CCM intercepte la demande
- Appel API Hetzner pour créer le LB
- IP publique assignée automatiquement
- Traefik exposé sur Internet

**Alternative pour autres clouds:** AWS ELB, GCP Load Balancer, Azure LB, ou MetalLB (on-premise)

---

## Slide 13 - Sécurité et Bonnes Pratiques

**Secrets Management**

- Token API Hetzner stocké en Secret Kubernetes
- Pas de credentials dans Git
- GitHub Secrets pour les workflows

**Network Policies**

- Calico permet de définir des règles réseau
- Isolation entre namespaces

**RBAC (Role-Based Access Control)**

- ArgoCD avec permissions limitées
- Principe du moindre privilège

**Updates**

- Images taggées avec SHA (pas `latest`)
- Traçabilité complète des versions

---

## Slide 14 - Monitoring et Observabilité

**État Actuel**

- Logs Kubernetes natifs
- Interface ArgoCD pour visualiser les déploiements
- GitHub Actions pour le CI/CD

**Améliorations Possibles**

1. **Monitoring**: Prometheus + Grafana
   - Métriques CPU/RAM/Réseau
   - Alertes automatiques

2. **Logging Centralisé**: ELK Stack ou Loki
   - Agrégation des logs
   - Recherche et analyse

3. **Tracing**: Jaeger ou Zipkin
   - Traçabilité des requêtes

4. **Health Checks**: Liveness & Readiness probes

---

## Slide 15 - Haute Disponibilité

**Configuration Actuelle**

- 2 VPS (multi-node)
- 1 replica pour chat-server
- 1 replica pour Redis

**Amélioration: Scalabilité Horizontale**

1. **Horizontal Pod Autoscaler (HPA)**
   - Scale automatique selon CPU/RAM
   - Min/Max replicas

2. **Redis en Cluster**
   - Redis Sentinel ou Redis Cluster
   - Haute disponibilité

3. **Multi-region**
   - Déploiement sur plusieurs datacenters
   - Geo-replication

---

## Slide 16 - Rollback et Disaster Recovery

**Rollback Facile avec GitOps**

1. **Via Git:**
   ```bash
   git revert HEAD
   git push
   ```
   ArgoCD applique automatiquement l'ancienne version

2. **Via ArgoCD Interface:**
   - Sélectionner un commit précédent
   - Clic sur "Sync"

3. **Via Kubernetes:**
   ```bash
   kubectl rollout undo deployment/chat-server -n chat-app
   ```

**Backup des Données**

- Velero pour backup complet du cluster
- Snapshots des volumes Hetzner
- Export Redis périodique

---

## Slide 17 - Coûts et Performances

**Infrastructure Hetzner**

- 2 VPS 4GB RAM: ~8€/mois chacun
- LoadBalancer: ~5€/mois
- Volume 1Gi HDD: ~0.05€/mois

**Total: ~21€/mois**

**Performances**

- Latence faible (Helsinki)
- Rolling updates sans downtime
- Redis persistant
- HTTPS avec certificats valides

**Comparaison AWS/GCP:** 3-5x plus cher pour équivalent

---

## Slide 18 - Défis Rencontrés

**1. LoadBalancer ne se provisionne pas**
- **Cause:** Région non spécifiée
- **Solution:** Annotation `load-balancer.hetzner.cloud/location=hel1`

**2. Redis redémarre en boucle**
- **Cause:** Volume non encore attaché
- **Solution:** Patience, c'est normal (provisionnement asynchrone)

**3. Certificat SSL non généré**
- **Cause:** Cert-Manager pas encore installé
- **Solution:** Vérifier l'ordre de déploiement ArgoCD

**4. Image Docker pas à jour**
- **Cause:** Cache GitHub Actions
- **Solution:** Tag basé sur SHA Git

---

## Slide 19 - Compétences Acquises

**DevOps & SRE**

- Infrastructure as Code (IaC)
- GitOps avec ArgoCD
- CI/CD avec GitHub Actions
- Gestion de secrets

**Kubernetes**

- Déploiement d'applications
- Gestion du stockage persistant
- Networking (CNI, Ingress, LoadBalancer)
- RBAC et sécurité

**Cloud Provider**

- Hetzner Cloud API
- Cloud Controller Manager
- CSI Driver

**Outils**

- Helm (templating)
- Kubectl
- Docker
- Git

---

## Slide 20 - Architecture Finale - Vue Complète

```
Internet
    ↓ HTTPS
Hetzner LoadBalancer (IP publique)
    ↓
Traefik Ingress Controller
    ↓
Service chat-server (ClusterIP)
    ↓
Pod chat-server (app)
    ↓ Redis connection
Service Redis (ClusterIP)
    ↓
Pod Redis
    ↓
PersistentVolume (Hetzner Cloud Volume 1Gi)
```

**Composants Externes:**
- DuckDNS (DNS)
- Let's Encrypt (SSL)
- GitHub Container Registry (images)
- GitHub (source code + GitOps)

---

## Slide 21 - Démonstration

**Live Demo**

1. **Application en Production**
   - https://sylvain-chat.duckdns.org
   - Certificat SSL valide

2. **Interface ArgoCD**
   - Visualisation des applications
   - État de synchronisation

3. **Workflow CI/CD**
   - Push de code
   - GitHub Actions en action
   - Déploiement automatique

4. **Kubernetes Dashboard**
   - Pods, Services, Ingress
   - Volumes persistants

---

## Slide 22 - Évolutions Futures

**Court Terme**

- Monitoring avec Prometheus/Grafana
- Alerting automatique
- Health checks plus robustes

**Moyen Terme**

- Autoscaling (HPA)
- Redis en haute disponibilité
- Backup automatique avec Velero

**Long Terme**

- Multi-region deployment
- Service Mesh (Istio/Linkerd)
- Observabilité avancée (tracing)
- Blue/Green ou Canary deployments

---

## Slide 23 - Conclusion

**Réalisations**

✅ Application web déployée en production
✅ Infrastructure complètement automatisée
✅ Pipeline CI/CD de bout en bout
✅ SSL/TLS automatique
✅ Haute disponibilité (2 nœuds)
✅ GitOps pour traçabilité et rollback facile
✅ Coût maîtrisé (~21€/mois)

**Bénéfices**

- Déploiement en production en quelques minutes
- Rollback instantané en cas de problème
- Infrastructure reproductible
- Évolutif et maintenable

---

## Slide 24 - Questions ?

**Ressources**

- GitHub: [ynov-k8s-orchestration](https://github.com/Sipixer/ynov-k8s-orchestration)
- Application: [sylvain-chat.duckdns.org](https://sylvain-chat.duckdns.org)
- Documentation: README.md & WALKTHROUGH.md

**Contact**

- Repository: https://github.com/Sipixer

**Merci pour votre attention !**
