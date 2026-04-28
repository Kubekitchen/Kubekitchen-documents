# KubeKitchen — End-to-End Project Story

> *"Tell me about your project from start to finish."*

---

## The Problem We Solved

We set out to build **KubeKitchen** — a production-grade, cloud-native food delivery platform — not just as an application, but as a complete engineering system that demonstrates how modern software is built, secured, deployed, and operated at scale. The goal was to connect every layer of the stack: from a developer writing a single line of code, all the way to that change safely reaching end users in production with zero downtime, full audit trail, and automated security gates along the way.

---

## Chapter 1 — The Application (What We Built)

KubeKitchen is a food delivery platform decomposed into **five independent microservices**, each owning its own domain:

| Service | Port | Responsibility |
|---|---|---|
| `auth-service` | 4001 | User registration, login, JWT token issuance |
| `restaurant-service` | 4002 | Restaurant CRUD and management |
| `menu-service` | 4003 | Menu items per restaurant |
| `order-service` | 4004 | Order lifecycle — create, track, update |
| `frontend` | 8080 | React/Next.js customer-facing UI |

Every backend service is written in **Node.js** and communicates over HTTP REST. Authentication is stateless — we use **JWT (JSON Web Tokens)**. When a user logs in, the `auth-service` issues a signed JWT token. Every subsequent API request to other services carries this token in the Authorization header. Services verify the token signature using a shared `JWT_SECRET` — no shared session state, no database round-trips for auth.

Each service has its **own dedicated MongoDB instance** — this is the **database-per-service pattern**. It enforces true independence: the `menu-service` cannot accidentally query the `order-service`'s database. Each service owns its data entirely. If the `order-service` database goes down, customers can still browse restaurants and menus.

The frontend communicates with backends exclusively through the API Gateway (Nginx Ingress) — never directly to a pod IP.

---

## Chapter 2 — Infrastructure (Where It Runs)

We self-managed the entire Kubernetes cluster on **AWS EC2** using `kubeadm` — not EKS, not a managed service. This was a deliberate choice to understand and control every layer of the infrastructure.

### Nodes

| Node | Instance | Role |
|---|---|---|
| Control Plane | t3.medium | API server, etcd, scheduler, controller-manager |
| Worker Node 1 | t3.medium | Runs all application workloads |
| HAProxy VM | t3.micro | Public-facing load balancer |

### Networking — The Traffic Path

```
User Browser
    │
    ▼
AWS Elastic IP: 54.88.142.144  ← Stable public IP (survives EC2 restarts)
    │
    ▼
HAProxy (t3.micro VM)          ← Layer 7 proxy, health checks, routing
    │  Port 80  → Worker1:80   (Nginx Ingress)
    │  Port 32001 → ControlPlane:32001 (ArgoCD UI)
    ▼
Nginx Ingress Controller       ← Path-based routing, runs as DaemonSet on Worker1
    │  /          → frontend:8080
    │  /api/auth  → auth-service:4001
    │  /api/restaurants → restaurant-service:4002
    │  /api/menu  → menu-service:4003
    │  /api/orders → order-service:4004
    ▼
Kubernetes ClusterIP Services  ← Stable virtual IPs inside the cluster
    ▼
Application Pods               ← The actual running containers
```

**HAProxy** acts as the production entry point. It performs health checks — if the Nginx Ingress becomes unhealthy, HAProxy stops routing traffic to it. It also exposes a real-time stats dashboard on port `8404` for monitoring active connections, request rates, and backend health. HAProxy is configured via `/etc/haproxy/haproxy.cfg` and enabled as a `systemd` service so it auto-starts on reboot.

**Nginx Ingress Controller** is deployed as a DaemonSet pinned to Worker Node 1 via `nodeSelector`. It handles path-based routing — all traffic entering the cluster on port 80 is routed to the correct microservice based on the URL path.

**Pod Networking** is handled by **Weave CNI**. Every pod gets a unique IP, and pods can communicate with each other across the cluster using these IPs via Weave's overlay network. Kubernetes CoreDNS resolves service names like `auth-service.kubekitchen-dev.svc.cluster.local` to the correct ClusterIP.

### Persistent Storage — NFS

MongoDB is stateful — it needs its data to survive pod restarts. We use an **NFS-backed StorageClass** (`kubekitchen-nfs`). When a MongoDB StatefulSet is created, Kubernetes automatically provisions a **PersistentVolumeClaim (PVC)** from the NFS server. The MongoDB pod mounts this PVC. If the pod crashes and is rescheduled to a different node, it reattaches to the same NFS volume and picks up exactly where it left off. Data durability is guaranteed independently of the pod lifecycle.

---

## Chapter 3 — Containerisation (How We Package It)

Every microservice is containerised using **Docker with multi-stage builds**. This is a critical production practice:

```dockerfile
# Stage 1: Builder — installs ALL dependencies including devDependencies
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci                    # Reproducible install
COPY . .
RUN npm run build             # Compile TypeScript / bundle assets

# Stage 2: Runtime — only production artifacts, no build tools
FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER 1001                     # Non-root for security
EXPOSE 4001
CMD ["node", "dist/index.js"]
```

**Why multi-stage?** The builder stage may contain compilers, test frameworks, and dev dependencies — hundreds of megabytes of code that should never ship to production. The final image only contains the runtime artifact. This reduces image size by 60-70% and dramatically reduces the attack surface.

All images are published to **Docker Hub** under `secretpower/kubekitchen-<service>` with two tags:
- `v1.0.<run_number>` — immutable, pinned to a specific CI run (used for rollback)
- `dev-latest` — mutable, always points to the latest dev build

---

## Chapter 4 — Security in CI (Shift-Left Security)

Security is not an afterthought — it is the **first job that runs** in every CI pipeline.

### SAST — Static Application Security Testing (Semgrep)

Before any Docker image is built, **Semgrep** scans the source code for security vulnerabilities — SQL injection patterns, hardcoded credentials, insecure JWT configurations, XSS vulnerabilities, and React-specific security anti-patterns. Results are uploaded as SARIF reports to the **GitHub Security tab**, giving developers a permanent, searchable record of every finding.

```yaml
- name: Run Semgrep SAST scan
  uses: semgrep/semgrep-action@v1
  with:
    publishToken: ${{ secrets.SEMGREP_APP_TOKEN }}
    generateSarif: "1"
```

### SCA — Software Composition Analysis (Trivy)

**Trivy** scans both the source dependencies (`package.json`) and the final Docker image for known CVEs (Common Vulnerabilities and Exposures). It checks against the NVD (National Vulnerability Database) and reports any `CRITICAL` or `HIGH` severity vulnerabilities. This catches the class of attack where a legitimate package contains a hidden malicious payload or an unpatched known exploit.

```yaml
- name: Scan Docker image with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'secretpower/kubekitchen-auth-service:v1.0.46'
    severity: 'CRITICAL,HIGH'
```

---

## Chapter 5 — CI Pipeline (GitHub Actions)

We have **five independent CI pipelines**, one per microservice. They are triggered by path filters — pushing to `services/auth-service/**` only triggers the auth pipeline, not the others. This means each service is independently buildable, testable, and deployable.

### The Four Jobs

```
Push to main
    │
    ├─ Job 1: security-scan
    │         Semgrep SAST + Trivy SCA
    │         (Runs in parallel — no dependency on build)
    │
    ├─ Job 2: build-and-push
    │         Needs: security-scan ✅
    │         docker build (multi-stage)
    │         docker push → secretpower/kubekitchen-auth-service:v1.0.46
    │         docker push → secretpower/kubekitchen-auth-service:dev-latest
    │
    ├─ Job 3: approval (production gate)
    │         Needs: build-and-push ✅
    │         GitHub Environment: "production" (requires manual approval)
    │         Sends email notification to DevOps engineer
    │
    └─ Job 4: deploy (GitOps update)
              Needs: build-and-push + approval ✅
              Clones Kubekitchen-GitOps repo
              Uses yq to update: auth-service.image.tag = "v1.0.46" in values.yaml
              git commit + push with retry logic (handles concurrent CI runs)
```

The **Approval Gate** (Job 3) is a GitHub Environment with a required reviewer. No image goes to production without a human approving it. When approval is needed, the pipeline sends an email via SMTP with a direct link to the approval page.

The **Deploy Job** (Job 4) does NOT touch the Kubernetes cluster directly. It only updates a YAML file in a separate Git repository — the **GitOps repository**. This is the fundamental principle of GitOps: the Git repository is the single source of truth.

---

## Chapter 6 — CD Pipeline (ArgoCD + GitOps)

### The GitOps Repository

We maintain two separate Git repositories:
- `Kubekitchen-app` — application source code
- `Kubekitchen-GitOps` — Kubernetes manifests, Helm charts, configuration

This separation is deliberate. The application repo is modified by developers. The GitOps repo is modified by CI pipelines and reviewed by DevOps engineers. ArgoCD watches the GitOps repo, never the application repo.

### ArgoCD

**ArgoCD v3.3.8** runs in the `argocd` namespace. It continuously compares the **desired state** (GitOps repo) with the **actual state** (Kubernetes cluster). If they diverge — due to a CI pipeline pushing a new image tag, or someone manually changing a deployment — ArgoCD reconciles automatically.

We run **three ArgoCD Applications**:

| Application | Path | Namespace | Strategy |
|---|---|---|---|
| `kubekitchen-dev` | `kubekitchen-helm` | `kubekitchen-dev` | Rolling Update |
| `kubekitchen-prod` | `kubekitchen-helm` | `kubekitchen-prod` | Blue-Green (Argo Rollouts) |
| `kubekitchen-sealed-secrets` | `sealed-secrets` | Both | Plain YAML |

### Sync Waves — Ordered Deployment

We use ArgoCD **sync waves** to enforce deployment order within a single sync:

```
Wave -1: Secrets (must exist before anything reads them)
Wave  0: ServiceAccount, Namespaces
Wave  1: MongoDB StatefulSets (databases must be ready before services)
Wave  2: Microservice Deployments / Rollouts
Wave  3: Frontend
PostSync: Seeder Job (seeds initial data after all services are up)
```

This guarantees that when ArgoCD syncs a fresh cluster from zero, services never start before their databases are ready.

---

## Chapter 7 — Helm (Packaging Kubernetes Manifests)

Rather than writing raw YAML for every environment, we use **Helm** with an **umbrella chart** architecture:

```
kubekitchen-helm/         ← Umbrella chart
├── Chart.yaml
├── values.yaml           ← Global defaults (used by both envs)
├── values-dev.yaml       ← Dev overrides (rolling update, 1 replica)
├── values-prod.yaml      ← Prod overrides (blue-green, 2 replicas)
└── charts/               ← Sub-charts (one per microservice)
    ├── auth-service/
    ├── restaurant-service/
    ├── menu-service/
    ├── order-service/
    ├── frontend/
    ├── mongodb/
    └── seeder/
```

The key architectural decision is the **`global.blueGreen` flag**. When `false` (dev), Helm renders a standard `apps/v1 Deployment`. When `true` (prod), Helm renders an `argoproj.io/v1alpha1 Rollout`. One Helm chart, two completely different deployment strategies, controlled by a single boolean.

```yaml
# The same deployment template handles both:
{{- if .Values.global.blueGreen }}
apiVersion: argoproj.io/v1alpha1
kind: Rollout
{{- else }}
apiVersion: apps/v1
kind: Deployment
{{- end }}
```

We also use **HPA (Horizontal Pod Autoscaler)** templates in each sub-chart — if CPU utilization crosses 70%, Kubernetes automatically scales up the number of pods, up to a configured maximum.

---

## Chapter 8 — Deployment Strategies

### Development — Rolling Update

In `kubekitchen-dev`, every CI push triggers ArgoCD to apply a rolling update:

```
OLD pod (v1.0.45) → RUNNING, serving traffic
NEW pod (v1.0.46) → Starting...
    │
    ▼ (health check passes)
NEW pod (v1.0.46) → RUNNING, now serving traffic
OLD pod (v1.0.45) → TERMINATED
```

Configuration: `maxSurge: 1, maxUnavailable: 0` — at no point are there zero running pods. Zero-downtime guaranteed. The entire process is automatic — no human intervention.

### Production — Blue-Green (Argo Rollouts)

In `kubekitchen-prod`, we use **Argo Rollouts v1.9.0** with a Blue-Green strategy. Blue-Green means we run two versions simultaneously, but only one receives live traffic:

```
BEFORE PROMOTION:
┌──────────────────────┐    ┌──────────────────────┐
│  BLUE (active svc)   │    │  GREEN (preview svc) │
│  v1.0.45             │    │  v1.0.46             │
│  100% live traffic   │    │  0% live traffic     │
│  ← customers here    │    │  ← testing here      │
└──────────────────────┘    └──────────────────────┘
         Rollout Status: ⏸️  Paused — awaiting promotion

AFTER PROMOTION:
┌──────────────────────┐    ┌──────────────────────┐
│  GREEN (now active)  │    │  BLUE (scaling down) │
│  v1.0.46             │    │  v1.0.45             │
│  100% live traffic ✅│    │  kept 30s, then gone │
└──────────────────────┘    └──────────────────────┘
```

The DevOps engineer tests the Green version via port-forward, and when satisfied, runs:
```bash
kubectl argo rollouts promote auth-service -n kubekitchen-prod
```

Argo Rollouts atomically switches the `active` service selector. Customers feel nothing — it is a single Kubernetes API call that changes which pods receive traffic. If something goes wrong, `kubectl argo rollouts undo` instantly reverts to Blue. Mean Time to Recovery (MTTR) is under 10 seconds.

---

## Chapter 9 — Secrets Management (Bitnami Sealed Secrets)

This is one of the most critical security components of the project. The problem: Kubernetes Secrets are just base64-encoded strings. If you commit a Secret YAML to Git, **anyone with repository access has your database passwords and JWT signing keys**.

**Sealed Secrets** solves this with asymmetric encryption:

```
Developer workflow:
1. Create plain Secret YAML locally (NEVER commit this)
2. kubeseal --format yaml < secret.yaml > sealed-secret.yaml
   └── Encrypts using the controller's PUBLIC key
3. Delete the plain secret file
4. git commit + push sealed-secret.yaml ← SAFE to commit
   └── Encrypted blob — useless without the controller's private key

In the cluster:
5. ArgoCD (kubekitchen-sealed-secrets app) applies the SealedSecret
6. Sealed Secrets controller (kube-system) decrypts using PRIVATE key
7. Creates a real Kubernetes Secret in the target namespace
8. Pods read the Secret via environment variables
```

We maintain:
- `sealed-secrets/sealed-secret-dev.yaml` → decrypts to `kubekitchen-secrets` in `kubekitchen-dev`
- `sealed-secrets/sealed-secret-prod.yaml` → decrypts to `kubekitchen-secrets` in `kubekitchen-prod`

The `kubekitchen-secrets` Secret contains:
```
JWT_SECRET, MONGO_AUTH_URI, MONGO_RESTAURANT_URI, MONGO_MENU_URI, MONGO_ORDER_URI
```

The encryption is **cluster-specific** — a SealedSecret encrypted with one cluster's controller key cannot be decrypted by another cluster. This is a feature: stolen Git repository contents are cryptographically useless without the cluster's private key.

---

## Chapter 10 — Kubernetes Security

### RBAC (Role-Based Access Control)

Every application pod runs under a dedicated **ServiceAccount** (`kubekitchen-sa`) with the **minimum permissions required** — it can only read Secrets and ConfigMaps in its own namespace. No pod has cluster-admin permissions. This follows the principle of least privilege.

### Non-Root Containers

All containers run as `USER 1001` (non-root). This means even if an attacker achieves Remote Code Execution inside a container, they cannot write to root-owned files on the node or escalate privileges.

### Namespace Isolation

We maintain complete environment separation:
- `kubekitchen-dev` — development, loose constraints, rolling updates
- `kubekitchen-prod` — production, strict resources, blue-green, sealed secrets
- `argocd` — GitOps controller, separate RBAC
- `ingress-nginx` — ingress controller
- `kube-system` — cluster system components including Sealed Secrets controller

---

## Chapter 11 — Observability

### Prometheus + Grafana

**Prometheus** is deployed in the cluster and scrapes metrics from:
- Node Exporter (CPU, memory, disk per node)
- kube-state-metrics (pod status, deployment health, replica counts)
- Application pods (custom metrics via `/metrics` endpoint)

**Grafana** connects to Prometheus as a data source and provides dashboards for:
- Pod CPU and memory utilization
- Request rates per service
- Error rates and latency percentiles
- MongoDB storage usage
- Node capacity and scheduling headroom

### HAProxy Stats

HAProxy exposes a real-time stats dashboard at `http://54.88.142.144:8404`. It shows active connections, request rates, response times, and backend health status per service — giving immediate visibility into the traffic layer without needing to log into the cluster.

### Headlamp

**Headlamp** provides a Kubernetes-native UI for cluster management — browsing pods, viewing logs, inspecting resources, and checking events — without needing `kubectl` access. It is particularly useful for developers who need visibility into the cluster without direct CLI access.

---

## Chapter 12 — Branching Strategy

We follow a **GitFlow-inspired** strategy:

| Branch | Purpose | CI Trigger |
|---|---|---|
| `main` | Production-ready code | Full CI + CD (builds + updates GitOps) |
| `feature/*` | New features in development | CI build only (no deploy) |
| `hotfix/*` | Emergency production fixes | Full CI, bypasses normal PR flow |

Every merge to `main` is protected by a Pull Request requiring at least one reviewer approval and all CI checks passing (including Semgrep and Trivy). The `main` branch is the single source of truth — everything deployed to any environment traces back to a specific commit on `main`.

---

## Chapter 13 — The Complete End-to-End Flow

Let me trace a single feature — "Fix the menu price calculation bug" — from developer's laptop to production:

```
Day 1 — Development:
  Priya branches: git checkout -b feature/fix-menu-price
  She fixes the bug in services/menu-service/routes/menu.js
  Local testing passes
  She opens a Pull Request to main

Day 1 — Code Review & CI:
  GitHub Actions triggers on the PR:
    ✅ Semgrep: No new security issues found
    ✅ Trivy: No critical CVEs in dependencies
    ✅ Jest: All unit tests pass
  Reviewer approves the PR
  Priya merges to main

Day 1 — CI Build (automatic, ~3 minutes):
  GitHub Actions triggers on push to main:
    ✅ Security scan passes
    ✅ Docker multi-stage build: secretpower/kubekitchen-menu-service:v1.0.47
    ✅ Trivy image scan: clean
    ✅ Push to Docker Hub (both :v1.0.47 and :dev-latest)
    📧 Approval email sent to DevOps engineer
    ⏸️  Pipeline waits for approval...

Day 1 — Approval & GitOps Update:
  DevOps engineer approves in GitHub
    ✅ Clone Kubekitchen-GitOps repo
    ✅ yq: menu-service.image.tag = "v1.0.47" in kubekitchen-helm/values.yaml
    ✅ git commit + push to GitOps repo

Day 1 — ArgoCD Detects Change (within 3 minutes):
  ArgoCD polls GitOps repo → detects values.yaml changed
  Renders Helm: blueGreen=false → apps/v1 Deployment
  Applies rolling update to kubekitchen-dev:
    → New pod (v1.0.47) starts
    → Health check passes
    → Old pod (v1.0.46) terminates
  kubekitchen-dev: ✅ Synced + Healthy

Day 2 — Production Promotion:
  DevOps engineer updates values-prod.yaml:
    menu-service.image.tag = "v1.0.47"
  git commit + push
  
  ArgoCD detects change → blueGreen=true → Argo Rollout
  Green pods (v1.0.47) start alongside Blue pods (v1.0.46)
  Rollout pauses: ⏸️

  DevOps engineer tests Green via port-forward:
    kubectl port-forward svc/menu-service-preview 4003:4003
    → Prices display correctly ✅

  Promotion:
    kubectl argo rollouts promote menu-service -n kubekitchen-prod
    Active service selector → Green pods
    Customers now on v1.0.47 ✅
    Blue pods (v1.0.46) scale down after 30 seconds

Total time from code merge to production: ~24 hours (includes human review gate)
Downtime: 0 seconds
```

---

## Summary — What Makes This Production-Grade

| Pillar | Implementation |
|---|---|
| **Security** | SAST (Semgrep), SCA (Trivy), JWT auth, Non-root containers, RBAC, Sealed Secrets, Namespace isolation |
| **Reliability** | Blue-Green deployments, Rolling updates, Health probes, Self-healing pods, PVC-backed databases |
| **Scalability** | HPA per service, Multi-replica prod, NFS storage, Stateless microservices |
| **Observability** | Prometheus metrics, Grafana dashboards, HAProxy stats, Headlamp UI, ArgoCD audit trail |
| **Automation** | Full CI/CD pipeline, GitOps reconciliation, Approval gates, Retry logic |
| **Auditability** | Every change traceable to a Git commit, ArgoCD history with timestamps and committers |
| **Zero Downtime** | maxUnavailable=0 rolling update (dev), Blue-Green with instant promotion (prod) |

> KubeKitchen is not just a food delivery app.
> It is a demonstration of how a real engineering team builds, secures, ships, and operates software in 2025.

---
*Built with: Node.js · React · MongoDB · Docker · Kubernetes (kubeadm) · Helm · GitHub Actions · ArgoCD · Argo Rollouts · Sealed Secrets · Prometheus · Grafana · HAProxy · Nginx Ingress · Weave CNI · NFS · AWS EC2 · Semgrep · Trivy · Headlamp*
