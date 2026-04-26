<div align="center">

# 🍽️ KubeKitchen

### Cloud-Native Food Delivery Platform on Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.34-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-3.3.8-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Helm](https://img.shields.io/badge/Helm-3.x-0F1689?style=for-the-badge&logo=helm&logoColor=white)](https://helm.sh/)
[![Docker](https://img.shields.io/badge/Docker-Hub-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://hub.docker.com/)
[![GitHub Actions](https://img.shields.io/badge/GitHub_Actions-CI%2FCD-2088FF?style=for-the-badge&logo=githubactions&logoColor=white)](https://github.com/features/actions)
[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/)
[![MongoDB](https://img.shields.io/badge/MongoDB-6.0-47A248?style=for-the-badge&logo=mongodb&logoColor=white)](https://www.mongodb.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

<br/>

> A production-grade, fully automated food delivery platform built on microservices,
> orchestrated with Kubernetes, deployed via GitOps, and designed for zero-downtime
> blue-green production releases.

<br/>

[🚀 Live Demo](#-live-demo) • [🏗️ Architecture](#️-architecture) • [⚡ Quick Start](#-quick-start) • [📖 Docs](#-documentation) • [🤝 Contributing](#-contributing)

<br/>

</div>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Live Demo](#-live-demo)
- [Architecture](#️-architecture)
- [Microservices](#-microservices)
- [Infrastructure](#️-infrastructure)
- [CI/CD Pipeline](#-cicd-pipeline)
- [GitOps with ArgoCD](#-gitops-with-argocd)
- [Deployment Strategies](#-deployment-strategies)
- [Secrets Management](#-secrets-management)
- [Quick Start](#-quick-start)
- [Project Structure](#-project-structure)
- [Technology Stack](#️-technology-stack)
- [Observability](#-observability)
- [Documentation](#-documentation)
- [Contributing](#-contributing)
- [License](#-license)

---

## 🌟 Overview

**KubeKitchen** is a cloud-native food delivery application that demonstrates
real-world production engineering practices. It is fully decomposed into
domain-driven microservices, each independently deployable, scalable, and
with its own dedicated database.

### ✨ Key Highlights

| Feature | Implementation |
|---------|---------------|
| 🔄 **GitOps** | ArgoCD continuously syncs cluster state from Git |
| 🚀 **Zero-Downtime Deployments** | Rolling updates (dev) + Blue-Green (prod) |
| 🔐 **Encrypted Secrets in Git** | Bitnami Sealed Secrets — safe to commit |
| 📦 **Single-Command Deploy** | Helm umbrella chart packages all 5 services |
| ⚙️ **Fully Automated CI** | GitHub Actions: lint → test → build → push → deploy |
| 📊 **Auto-Scaling** | HPA scales pods on CPU metrics automatically |
| 💾 **Persistent Storage** | NFS-backed PVCs for MongoDB — survives pod restarts |
| 🌐 **Production Traffic Control** | HAProxy + Elastic IP for stable public entry |

---

## 🔴 Live Demo

| Service | URL |
|---------|-----|
| 🍽️ **Application** | [http://54.88.142.144](http://54.88.142.144) |
| 🔁 **ArgoCD UI** | [http://54.88.142.144:32001](http://54.88.142.144:32001) |
| 📊 **HAProxy Stats** | [http://54.88.142.144:8404/stats](http://54.88.142.144:8404/stats) |

> **Note:** The cluster runs on AWS EC2 (t3.medium). If the demo is not
> available, the instances may be stopped to manage costs.

---

## 🏗️ Architecture

### System Overview


---

## 🧩 Microservices

| Service | Language | Port | Responsibility | Docker Image |
|---------|----------|------|----------------|--------------|
| **auth-service** | Node.js | 4001 | JWT authentication, registration, login | `secretpower/kubekitchen-auth` |
| **restaurant-service** | Node.js | 4002 | Restaurant CRUD operations | `secretpower/kubekitchen-restaurant` |
| **menu-service** | Node.js | 4003 | Menu items per restaurant | `secretpower/kubekitchen-menu` |
| **order-service** | Node.js | 4004 | Order lifecycle management | `secretpower/kubekitchen-order` |
| **frontend** | Next.js | 8080 | Customer-facing React UI | `secretpower/kubekitchen-frontend` |

Each service:
- ✅ Has its **own dedicated MongoDB** instance (database-per-service pattern)
- ✅ Is independently **deployable and scalable**
- ✅ Has its **own CI/CD workflow**
- ✅ Has **HPA** configured for CPU-based auto-scaling
- ✅ Reads configuration from a shared `kubekitchen-secrets` Kubernetes Secret

---

## 🖥️ Infrastructure

### AWS Resources

┌────────────────────────────────────────────────────────────────────┐
│                         AWS INFRASTRUCTURE                          │
├──────────────────┬───────────────┬──────────────────┬─────────────┤
│ ROLE             │ INSTANCE TYPE │ PRIVATE IP       │ PURPOSE     │
├──────────────────┼───────────────┼──────────────────┼─────────────┤
│ Control Plane    │ t3.medium     │ 172.31.69.182    │ K8s master  │
│ Worker Node 1    │ t3.medium     │ 172.31.66.23     │ App workload│
│ HAProxy VM       │ t3.micro      │ 172.31.30.143    │ Public LB   │
│                  │               │ EIP: 54.88.142.144│            │
└──────────────────┴───────────────┴──────────────────┴─────────────┘

### Kubernetes Namespaces

| Namespace | Purpose |
|-----------|---------|
| `kubekitchen-dev` | Development environment (rolling updates) |
| `kubekitchen-prod` | Production environment (blue-green deployments) |
| `argocd` | GitOps controller |
| `ingress-nginx` | Nginx ingress controller |
| `argo-rollouts` | Progressive delivery controller |
| `kube-system` | Core Kubernetes + Sealed Secrets controller |

### Networking

- **CNI:** Weave Net (pod overlay networking)
- **Ingress:** ingress-nginx DaemonSet pinned to Worker Node 1
- **External Access:** HAProxy with AWS Elastic IP
- **Internal DNS:** CoreDNS (`<service>.<namespace>.svc.cluster.local`)

### Storage

- **Provider:** NFS Dynamic Provisioner
- **StorageClass:** `kubekitchen-nfs`
- **Usage:** MongoDB PersistentVolumeClaims (1Gi each)
- **Retention:** `Retain` policy — data survives pod deletion

---

## 🔄 CI/CD Pipeline

KubeKitchen uses **GitHub Actions** for Continuous Integration with a
separate workflow per service.

### Pipeline Stages
┌─────────┐    ┌──────────┐    ┌───────────────────┐    ┌───────────────┐
│  lint   │───▶│   test   │───▶│  build-and-push   │───▶│ update-gitops │
│         │    │          │    │                   │    │               │
│ ESLint  │    │  Jest    │    │ docker build      │    │ clone GitOps  │
│ code    │    │  unit    │    │ push :dev-<sha>   │    │ yq update tag │
│ quality │    │  tests   │    │ push :dev-latest  │    │ git push      │
└─────────┘    └──────────┘    └───────────────────┘    └───────────────┘


### Trigger Condition

```yaml
on:
  push:
    branches: [main]
    paths:
      - 'auth-service/**'   # Only triggers if this service changed


      Image Tagging Strategy
Tag	Example	Purpose
dev-<sha>	dev-a1b2c3d	Immutable — used by ArgoCD, enables rollback
dev-latest	dev-latest	Mutable — convenience for local dev
v1.x.x	v1.2.0	Production release tag
Required GitHub Secrets
Secret	Purpose
DOCKER_USERNAME	Docker Hub username
DOCKER_PASSWORD	Docker Hub access token
GH_PAT	Personal Access Token to write to GitOps repo
🚀 GitOps with ArgoCD
ArgoCD monitors the GitOps repository and automatically syncs the cluster
state to match what's declared in Git.

Applications
App Name	Environment	Values Files	Sync Policy
kubekitchen-dev	kubekitchen-dev	values.yaml + values-dev.yaml	Auto-sync + Prune
kubekitchen-prod	kubekitchen-prod	values.yaml + values-prod.yaml	Auto-sync + Prune
Sync Wave Order
Wave -1 → SealedSecrets (secrets must exist first)
Wave 0 → ConfigMaps (config before apps)
Wave 1 → MongoDB (database before application)
Wave 2 → Services/Pods (application layer)
Wave 3 → Seeder Job (PostSync data seeding)

ArgoCD Access
Bash

# Login via CLI
argocd login 54.88.142.144:32001 --username admin --insecure

# Sync an application manually
argocd app sync kubekitchen-dev

# Check application health
argocd app get kubekitchen-dev
🎯 Deployment Strategies
Development — Rolling Update
Trigger: New image tag detected in values-dev.yaml by ArgoCD
↓
[Pod v1] [Pod v1] (initial state)
[Pod v1] [Pod v1] [Pod v2] (new pod created, maxSurge: 1)
[Pod v1] [Pod v2] (old pod terminated, maxUnavailable: 0)
[Pod v2] [Pod v2] [Pod v2] (another new pod)
[Pod v2] [Pod v2] (fully rolled out)

Config: maxSurge: 1 | maxUnavailable: 0 | Zero downtime guaranteed

Production — Blue-Green (Argo Rollouts)
New image pushed to prod values
↓
GREEN pods deployed → preview-service (no user traffic yet)
↓
Engineer tests GREEN via preview service port-forward
↓
kubectl argo rollouts promote <name> -n kubekitchen-prod
↓
Active service switches → GREEN pods (instant traffic shift)
↓
BLUE pods kept 30s (scaleDownDelaySeconds) then terminated

Controller: Argo Rollouts v1.9.0 | autoPromotionEnabled: false (manual gate)

Bash

# Check rollout status
kubectl argo rollouts get rollout <name> -n kubekitchen-prod --watch

# Promote (go live with new version)
kubectl argo rollouts promote <name> -n kubekitchen-prod

# Abort (instant rollback to blue)
kubectl argo rollouts abort <name> -n kubekitchen-prod

🔐 Secrets Management
KubeKitchen uses Bitnami Sealed Secrets to safely store encrypted
credentials in Git.

Why Sealed Secrets?
❌ Plain Kubernetes Secret in Git → base64 decoded in seconds → UNSAFE

✅ SealedSecret in Git → asymmetrically encrypted with cluster public key
→ Only the in-cluster controller can decrypt it → SAFE TO COMMIT

Secrets Stored
Key	Used By
JWT_SECRET	auth-service
MONGO_AUTH_URI	auth-service → mongodb-auth
MONGO_RESTAURANT_URI	restaurant-service → mongodb-restaurant
MONGO_MENU_URI	menu-service → mongodb-menu
MONGO_ORDER_URI	order-service → mongodb-order
Creating a Sealed Secret
Bash

# 1. Fetch cluster public key
kubeseal --fetch-cert \
  --controller-namespace kube-system \
  --controller-name sealed-secrets-controller \
  > pub-cert.pem

# 2. Seal your secret
kubeseal --cert pub-cert.pem --format yaml \
  < raw-secret.yaml > sealed-secrets/sealed-secret-dev.yaml

# 3. Commit sealed secret (safe!)
git add sealed-secrets/sealed-secret-dev.yaml
git commit -m "feat: update sealed secret"
git push
⚠️ Never commit the raw raw-secret.yaml file.
Sealed Secrets are cluster-specific — a secret sealed for cluster A
cannot be decrypted by cluster B.

⚡ Quick Start
Prerequisites
Bash

# Required tools
kubectl    >= 1.28
helm       >= 3.12
argocd     >= 2.9
kubeseal   >= 0.27.0
docker     >= 24.0
1. Clone the Repositories
Bash

# Application code
git clone https://github.com/Kubekitchen/Kubekitchen.git
cd Kubekitchen

# GitOps manifests (separate repo)
git clone https://github.com/Kubekitchen/Kubekitchen-GitOps.git
2. Bootstrap the Cluster (kubeadm)
Bash

# On Control Plane
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<control-plane-ip>

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

# Install Weave CNI
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# On Worker Node
sudo kubeadm join <control-plane-ip>:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
3. Install Platform Components
Bash

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.8/manifests/install.yaml

# Install Argo Rollouts
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts \
  -f https://github.com/argoproj/argo-rollouts/releases/download/v1.9.0/install.yaml

# Install Sealed Secrets
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.27.0/controller.yaml

# Install ingress-nginx (pinned to Worker1)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
4. Deploy via ArgoCD
Bash

# Connect GitOps repo to ArgoCD
argocd repo add https://github.com/Kubekitchen/Kubekitchen-GitOps.git \
  --username <github-user> \
  --password <PAT>

# Create ArgoCD Applications
kubectl apply -f argocd-apps/kubekitchen-dev.yaml
kubectl apply -f argocd-apps/kubekitchen-prod.yaml

# Trigger initial sync
argocd app sync kubekitchen-dev
5. Verify Deployment
Bash

# Check all pods are running
kubectl get pods -n kubekitchen-dev

# Check ingress rules
kubectl get ingress -n kubekitchen-dev

# Test the API
curl http://54.88.142.144/api/auth/health
curl http://54.88.142.144/api/restaurants
📁 Project Structure
Kubekitchen/ ← Application Repository
├── auth-service/
│ ├── src/
│ ├── tests/
│ ├── Dockerfile
│ └── package.json
├── restaurant-service/
├── menu-service/
├── order-service/
├── frontend/
└── .github/
└── workflows/
├── ci-auth-service.yml
├── ci-restaurant-service.yml
├── ci-menu-service.yml
├── ci-order-service.yml
└── ci-frontend.yml

Kubekitchen-GitOps/ ← GitOps Repository (separate repo)
├── kubekitchen-helm/ ← Umbrella Helm Chart
│ ├── Chart.yaml
│ ├── values.yaml ← Global defaults
│ ├── values-dev.yaml ← Dev image tags (updated by CI)
│ ├── values-prod.yaml ← Prod config (blueGreen: true)
│ ├── templates/
│ └── charts/ ← Sub-charts
│ ├── auth-service/
│ │ ├── Chart.yaml
│ │ ├── values.yaml
│ │ └── templates/
│ │ ├── _helpers.tpl
│ │ ├── deployment.yaml ← Conditional: Deployment or Rollout
│ │ ├── service.yaml
│ │ └── hpa.yaml
│ ├── restaurant-service/ ← Same structure
│ ├── menu-service/
│ ├── order-service/
│ ├── frontend/
│ ├── mongodb/ ← StatefulSet + PVC + headless service
│ └── seeder/ ← PostSync Job
├── sealed-secrets/
│ ├── sealed-secret-dev.yaml
│ └── sealed-secret-prod.yaml
└── argocd-apps/
├── kubekitchen-dev.yaml
└── kubekitchen-prod.yaml

🛠️ Technology Stack
<div align="center">
Category	Technology	Version
Orchestration	Kubernetes (kubeadm)	1.34
Package Manager	Helm	3.x
GitOps	ArgoCD	3.3.8
Progressive Delivery	Argo Rollouts	1.9.0
Secrets	Bitnami Sealed Secrets	0.27.0
CI/CD	GitHub Actions	—
Registry	Docker Hub	—
CNI	Weave Net	2.8.1
Ingress	ingress-nginx	Latest
Load Balancer	HAProxy	2.x
Database	MongoDB	6.0
Storage	NFS Dynamic Provisioner	—
Cloud	AWS EC2	—
Backend	Node.js / Express	20 LTS
Frontend	Next.js / React	Latest
Monitoring	Prometheus + Grafana	—
</div>
📊 Observability
Metrics
Prometheus scrapes pod metrics via annotations
Grafana dashboards for CPU, memory, request rates
Access Points
Bash

# HAProxy Stats (request rates, backend health, error %)
http://54.88.142.144:8404/stats
# Credentials: admin / kubekitchen2024

# ArgoCD UI (sync status, resource health, deployment history)
http://54.88.142.144:32001
Useful Debug Commands
Bash

# Check pod health
kubectl get pods -n kubekitchen-dev -o wide

# Stream logs
kubectl logs -f deployment/kubekitchen-auth-service -n kubekitchen-dev

# Resource usage
kubectl top pods -n kubekitchen-dev
kubectl top nodes

# Kubernetes events (best for debugging)
kubectl get events -n kubekitchen-dev \
  --sort-by='.lastTimestamp'

# Check endpoints (debug service-to-pod routing)
kubectl get endpoints -n kubekitchen-dev
📖 Documentation
For deep technical reference, see the full KubeKitchen Technical Study Guide
which covers:

📐 Full architecture diagrams
🔧 Component deep-dives (Helm, ArgoCD, Rollouts, Secrets)
📝 Complete commands cheatsheet
🐛 Troubleshooting guide (5 real issues we solved)
🎯 End-to-end deployment walkthrough
🎤 Interview preparation Q&A
🐛 Known Issues & Solutions
Issue	Root Cause	Solution
Chart.yaml not found on Linux	Case sensitivity: chart.yaml vs Chart.yaml	Two-step git mv rename
Service endpoints <none>	3-label selector vs 1-label on pod	Use single app: <name> selector
Deployment update rejected	spec.selector is immutable	Delete + recreate Deployment
Ingress port conflict	DaemonSet scheduling on wrong node	nodeSelector to Worker1
Pods stuck Pending	CPU overcommit on t3.medium	Reduced requests: 100m → 30m
🤝 Contributing
Contributions are welcome! Please follow these steps:

Bash

# 1. Fork the repository
# 2. Create a feature branch
git checkout -b feature/your-feature-name

# 3. Make your changes and test
npm test

# 4. Commit with conventional commits
git commit -m "feat: add restaurant search endpoint"

# 5. Push and open a Pull Request
git push origin feature/your-feature-name
Commit Convention
text

feat:     New feature
fix:      Bug fix
ci:       CI/CD changes
docs:     Documentation
refactor: Code refactoring
chore:    Maintenance


