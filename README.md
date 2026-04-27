# lavenbloom-charts

> **Runbook & Operator Walkthrough** — Helm charts and Kubernetes infrastructure for the Lavenbloom microservices platform.

---

## Table of Contents

1. [Overview](#overview)
2. [Platform Architecture](#platform-architecture)
3. [Repository Structure](#repository-structure)
4. [Prerequisites](#prerequisites)
5. [Environment Strategy — Dev vs Prod](#environment-strategy--dev-vs-prod)
6. [Deployment Walkthrough — Dev Cluster](#deployment-walkthrough--dev-cluster)
7. [Deployment Walkthrough — Prod Cluster](#deployment-walkthrough--prod-cluster)
8. [ArgoCD GitOps Setup](#argocd-gitops-setup)
9. [Network Policy — Zero-Trust Model](#network-policy--zero-trust-model)
10. [Gateway Routes](#gateway-routes)
11. [Database Init Jobs](#database-init-jobs)
12. [StatefulSets and Persistence](#statefulsets-and-persistence)
13. [Secrets Management](#secrets-management)
14. [Verification Checklist](#verification-checklist)
15. [Upgrading a Service](#upgrading-a-service)
16. [Troubleshooting](#troubleshooting)

---

## Overview

`lavenbloom-charts` is the **GitOps source of truth** for all Kubernetes resources in the Lavenbloom platform. It contains:

- **Helm charts** for 5 microservices and their databases
- **Infrastructure charts** for namespaces, network policies, Redis, and the Envoy Gateway
- **ArgoCD ApplicationSets** that watch this repo and automatically sync both dev and prod clusters
- **Per-environment values files** (`values-dev.yaml` / `values-prod.yaml`) that hold environment-specific image tags and connection parameters

> CI/CD pipelines in service repos update `values-dev.yaml` / `values-prod.yaml` here via `cd-template.yml`. ArgoCD detects the commit and deploys automatically. No manual `kubectl apply` is required in normal operation.

---

## Platform Architecture

```
                        ┌──────────────────────────────────────────────────────┐
                        │              Kubernetes Cluster                       │
                        │                                                        │
  Internet ────────────▶│  ┌──────────────┐  Namespace: gateway                │
                        │  │ Envoy Gateway│  (kgateway / Gateway API)           │
                        │  │   NodePort   │                                      │
                        │  └──────┬───────┘                                     │
                        │         │ HTTPRoutes (path-based)                      │
                        │   ┌─────┼───────────────────────────────────┐         │
                        │   │     │         Namespace: backend          │         │
                        │   │ ┌───▼───────┐ ┌──────────┐ ┌─────────┐ │         │
                        │   │ │auth-svc   │ │habit-svc │ │journal  │ │         │
                        │   │ │:8000      │ │:8000     │ │svc:8000 │ │         │
                        │   │ └───────────┘ └──────────┘ └─────────┘ │         │
                        │   │ ┌──────────────────┐ ┌───────────────┐ │         │
                        │   │ │notification-svc  │ │  Redis :6379  │ │         │
                        │   │ │:8000 + worker    │ │  (StatefulSet)│ │         │
                        │   │ └──────────────────┘ └───────────────┘ │         │
                        │   └────────────────────────────────────────┘         │
                        │                                                        │
                        │   ┌────────────────────────────────────────┐          │
                        │   │         Namespace: db                   │          │
                        │   │ auth-db  habit-db  journal-db  notif-db│          │
                        │   │ (PostgreSQL StatefulSets, PVCs: nfs)   │          │
                        │   └────────────────────────────────────────┘          │
                        │                                                        │
                        │   ┌──────────────┐                                    │
                        │   │ Namespace:   │                                    │
                        │   │ frontend     │                                    │
                        │   │ nginx:80     │                                    │
                        │   └──────────────┘                                    │
                        └──────────────────────────────────────────────────────┘
```

### Namespaces

| Namespace | Purpose |
|---|---|
| `gateway` | Envoy Gateway controller pods and GatewayParameters |
| `backend` | All 4 microservice Deployments + Redis StatefulSet + notification worker |
| `db` | All 4 PostgreSQL StatefulSets and their PVCs |
| `frontend` | Frontend Nginx Deployment |

### Services

| Service | Env Vars | Database | Port | K8s Namespace |
|---|---|---|---|---|
| `auth-service` | `POSTGRES_URI`, `JWT_SECRET` | `auth_db` | `8000` | `backend` |
| `habit-service` | `POSTGRES_URI`, `JWT_SECRET`, `REDIS_URI` | `habit_db` | `8000` | `backend` |
| `journal-service` | `POSTGRES_URI`, `JWT_SECRET` | `journal_db` | `8000` | `backend` |
| `notification-service` | `POSTGRES_URI`, `REDIS_URI`, `JWT_SECRET` | `notification_db` | `8000` | `backend` |
| `notification-worker` | Same as notification-service | — | — | `backend` |
| `frontend` | `REACT_APP_API_URL` | — | `80` | `frontend` |

---

## Repository Structure

```
lavenbloom-charts/
├── argocd/
│   ├── projects/
│   │   └── lavenbloom-project.yaml      # ArgoCD Project (RBAC scope)
│   └── environments/
│       ├── dev-appset.yaml              # ApplicationSet for dev cluster
│       └── prod-appset.yaml             # ApplicationSet for prod cluster
│
├── infrastructure/
│   ├── namespaces/                      # Namespace manifests
│   ├── gateway/                         # Envoy Gateway + GatewayParameters + HTTPRoutes
│   ├── network-policies/                # NetworkPolicy manifests (zero-trust)
│   └── redis/                           # Redis StatefulSet + Services + Secret
│       └── redis/                       # (storageClass: nfs, 1Gi PVC)
│
├── microservices/
│   ├── auth-service/
│   │   ├── templates/
│   │   │   ├── deployment.yaml
│   │   │   ├── service.yaml
│   │   │   ├── secret.yaml
│   │   │   └── db/                      # DB StatefulSet, Services, Init Job, Secret
│   │   ├── Chart.yaml
│   │   ├── values-dev.yaml
│   │   └── values-prod.yaml
│   ├── habit-service/        # Same structure
│   ├── journal-service/      # Same structure
│   ├── notification-service/ # + worker-deployment.yaml
│   └── frontend/
│       ├── templates/
│       │   ├── deployment.yaml
│       │   └── service.yaml
│       ├── Chart.yaml
│       ├── values-dev.yaml
│       └── values-prod.yaml
│
└── README.md
```

---

## Prerequisites

| Tool | Minimum version | Purpose |
|---|---|---|
| `kubectl` | 1.27+ | Kubernetes CLI |
| `helm` | 3.12+ | Chart templating and install |
| `argocd` CLI | 2.8+ | Managing ArgoCD Applications |
| Envoy Gateway | v1.0+ | Provides the `eg` GatewayClass |
| NFS StorageClass | — | PVC provisioning for PostgreSQL and Redis |

### Verify cluster readiness

```bash
kubectl cluster-info
kubectl get nodes
kubectl get storageclass   # Expect 'nfs' to be listed
kubectl get gatewayclass   # Expect 'eg' to be listed
```

---

## Environment Strategy — Dev vs Prod

The platform uses **two completely separate Kubernetes clusters** — one for dev, one for prod. This is stronger isolation than the namespace-level minimum requirement.

| Aspect | Dev | Prod |
|---|---|---|
| **Cluster** | Dedicated dev cluster | Dedicated prod cluster |
| **ArgoCD** | `dev-argocd` instance | `prod-argocd` instance |
| **Image tags** | `dev-{7-char-SHA}` | Semver (`v1.2.0`) |
| **Values file** | `values-dev.yaml` | `values-prod.yaml` |
| **Trigger** | Push to `develop` branch | GitHub Release created |
| **ArgoCD AppSet** | `dev-appset.yaml` | `prod-appset.yaml` |

### Values file structure

Each service chart uses per-environment values files. Example for `auth-service`:

**`values-dev.yaml`:**
```yaml
auth-service:
  image:
    repository: lavenbloom-auth-service
    tag: "dev-a3f9c12"
  secrets:
    postgresUri: "postgresql://authuser:authpassword@auth-db.db.svc.cluster.local:5432/auth_db"
    jwtSecret: "supersecretjwtkey"
```

**`values-prod.yaml`:**
```yaml
auth-service:
  image:
    repository: lavenbloom-auth-service
    tag: "1.0.0"
  secrets:
    postgresUri: "postgresql://authuser:authpassword@auth-db.db.svc.cluster.local:5432/auth_db"
    jwtSecret: "supersecretjwtkey"
```

---

## Deployment Walkthrough — Dev Cluster

> Run these commands against your **dev** cluster context.

### Step 1 — Set kubectl context

```bash
kubectl config use-context <dev-cluster-context>
kubectl config current-context   # verify
```

### Step 2 — Install namespaces

```bash
helm install namespaces ./infrastructure/namespaces
kubectl get namespaces | grep -E "gateway|backend|db|frontend"
```

### Step 3 — Install Redis

```bash
helm install redis ./infrastructure/redis -f microservices/habit-service/values-dev.yaml
kubectl get statefulset -n backend redis
kubectl get pvc -n backend
```

### Step 4 — Install the Gateway

```bash
helm install gateway ./infrastructure/gateway -f microservices/auth-service/values-dev.yaml
kubectl get gateway -n gateway
kubectl get httproute -n gateway
```

Expected HTTPRoutes: `/auth`, `/habits`, `/journal`, `/notifications`, `/`

### Step 5 — Apply Network Policies

```bash
helm install network-policies ./infrastructure/network-policies
kubectl get networkpolicies -A
```

Expected: 9 NetworkPolicy objects (default-deny + per-service allow rules).

### Step 6 — Install Microservices

```bash
helm install auth-service         ./microservices/auth-service         -f ./microservices/auth-service/values-dev.yaml
helm install habit-service        ./microservices/habit-service        -f ./microservices/habit-service/values-dev.yaml
helm install journal-service      ./microservices/journal-service      -f ./microservices/journal-service/values-dev.yaml
helm install notification-service ./microservices/notification-service -f ./microservices/notification-service/values-dev.yaml
helm install frontend             ./microservices/frontend             -f ./microservices/frontend/values-dev.yaml
```

### Step 7 — Verify all pods are running

```bash
kubectl get pods -n backend
kubectl get pods -n db
kubectl get pods -n frontend
kubectl get pods -n gateway
```

All pods should reach `Running` / `Completed` state. DB init Jobs will show `Completed`.

### Step 8 — Smoke test via Gateway

```bash
# Get Gateway NodePort (or LoadBalancer IP)
kubectl get svc -n gateway

# Test auth-service
curl http://<GATEWAY_IP>:<PORT>/auth/health
# {"status":"ok"}

# Test habit-service
curl http://<GATEWAY_IP>:<PORT>/habits/health
# {"status":"ok"}
```

---

## Deployment Walkthrough — Prod Cluster

```bash
# Switch to prod cluster
kubectl config use-context <prod-cluster-context>

# Install infrastructure (same as dev)
helm install namespaces ./infrastructure/namespaces
helm install redis ./infrastructure/redis -f microservices/habit-service/values-prod.yaml
helm install gateway ./infrastructure/gateway -f microservices/auth-service/values-prod.yaml
helm install network-policies ./infrastructure/network-policies

# Install microservices with prod values
helm install auth-service         ./microservices/auth-service         -f ./microservices/auth-service/values-prod.yaml
helm install habit-service        ./microservices/habit-service        -f ./microservices/habit-service/values-prod.yaml
helm install journal-service      ./microservices/journal-service      -f ./microservices/journal-service/values-prod.yaml
helm install notification-service ./microservices/notification-service -f ./microservices/notification-service/values-prod.yaml
helm install frontend             ./microservices/frontend             -f ./microservices/frontend/values-prod.yaml
```

---

## ArgoCD GitOps Setup

ArgoCD watches this repository and automatically applies changes when `values-dev.yaml` or `values-prod.yaml` is updated by the CI/CD pipeline.

### Step 1 — Install ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Available deployment/argocd-server -n argocd --timeout=120s
```

### Step 2 — Access ArgoCD UI

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Open: https://localhost:8080
# Username: admin
# Password:
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

### Step 3 — Apply the ArgoCD Project

```bash
kubectl apply -f argocd/projects/lavenbloom-project.yaml
```

### Step 4 — Apply the ApplicationSet

```bash
# Dev cluster:
kubectl apply -f argocd/environments/dev-appset.yaml -n argocd

# Prod cluster:
kubectl apply -f argocd/environments/prod-appset.yaml -n argocd
```

ArgoCD will create one Application per microservice chart and begin syncing.

### Step 5 — Verify Applications

```bash
argocd app list
# NAME                    CLUSTER    NAMESPACE  STATUS  HEALTH
# auth-service            in-cluster backend    Synced  Healthy
# habit-service           in-cluster backend    Synced  Healthy
# ...
```

### ArgoCD sync behavior

| Setting | Value | Effect |
|---|---|---|
| `selfHeal: true` | Enabled | Any manual `kubectl` change is reverted by ArgoCD |
| `prune: true` | Enabled | Resources removed from the chart are deleted from the cluster |
| Poll interval | 3 minutes | ArgoCD checks for new commits every 3 minutes |

---

## Network Policy — Zero-Trust Model

All namespaces have a **default-deny-all** ingress NetworkPolicy. Traffic is only permitted by explicit allow rules.

### Policy summary (9 NetworkPolicy objects)

| Policy | Source | Destination | Allowed |
|---|---|---|---|
| `default-deny-backend` | any | `backend` namespace | ❌ deny all |
| `default-deny-db` | any | `db` namespace | ❌ deny all |
| `default-deny-frontend` | any | `frontend` namespace | ❌ deny all |
| `allow-gateway-to-backend` | `gateway` namespace | all backend pods | ✅ |
| `allow-gateway-to-frontend` | `gateway` namespace | frontend pods | ✅ |
| `allow-auth-to-authdb` | `auth-service` pod | `auth-db` pod | ✅ |
| `allow-habit-to-habitdb` | `habit-service` pod | `habit-db` pod | ✅ |
| `allow-journal-to-journaldb` | `journal-service` pod | `journal-db` pod | ✅ |
| `allow-notif-to-notifdb` | `notification-*` pods | `notification-db` pod | ✅ |

### Key properties

- **Microservices cannot talk to each other** — `habit-service` cannot reach `auth-service` or `journal-service` directly
- **Redis is accessible only from backend** — Redis does not have a NodePort; only in-cluster pods with the correct label can connect
- **Databases are fully isolated** — `auth-db` only accepts connections from `auth-service`; `habit-db` only from `habit-service`, etc.

### Verify a policy is working

```bash
# This should succeed (gateway → backend)
kubectl exec -n gateway <gateway-pod> -- curl http://auth-service.backend.svc.cluster.local:8000/health

# This should fail (direct habit → auth cross-talk is denied)
kubectl exec -n backend <habit-pod> -- curl http://auth-service.backend.svc.cluster.local:8000/health
```

---

## Gateway Routes

The Kubernetes Gateway API (Envoy/kgateway) exposes all services on a single NodePort entry point. HTTPRoutes perform path-prefix matching and URL rewriting.

| Path Prefix | Target Service | Namespace | Port | URL Rewrite |
|---|---|---|---|---|
| `/auth` | `auth-service` | `backend` | `8000` | Strip `/auth` prefix |
| `/habits` | `habit-service` | `backend` | `8000` | Strip `/habits` prefix |
| `/journal` | `journal-service` | `backend` | `8000` | Strip `/journal` prefix |
| `/notifications` | `notification-service` | `backend` | `8000` | Strip `/notifications` prefix |
| `/` | `frontend` | `frontend` | `80` | — |

### Verify routing

```bash
# List all HTTPRoutes
kubectl get httproutes -A

# Describe a specific route
kubectl describe httproute auth-route -n gateway

# Test gateway routing
GATEWAY_IP=$(kubectl get svc -n gateway -o jsonpath='{.items[0].status.loadBalancer.ingress[0].ip}')
curl http://$GATEWAY_IP/auth/health
curl http://$GATEWAY_IP/habits/health
```

---

## Database Init Jobs

Each microservice chart includes a Kubernetes **Job** (`db-init-job.yaml`) deployed as a Helm `post-install` hook. The job:

1. Waits for the PostgreSQL pod to be ready (`pg_isready` loop)
2. Creates the application database (`CREATE DATABASE IF NOT EXISTS`)
3. Creates the application user (`CREATE USER IF NOT EXISTS`)
4. Grants all privileges on the database to the user

**Key properties:**
- Runs only once per `helm install` (post-install hook)
- **Idempotent** — uses `IF NOT EXISTS` and `|| true` to safely re-run
- Auto-cleans up after completion (`hook-delete-policy: hook-succeeded`)

### Check init job status

```bash
kubectl get jobs -n db
# NAME              COMPLETIONS   DURATION
# auth-db-init      1/1           8s
# habit-db-init     1/1           6s
# journal-db-init   1/1           7s
# notif-db-init     1/1           7s

# View job logs if it fails
kubectl logs job/auth-db-init -n db
```

### Re-run a failed init job manually

```bash
kubectl delete job auth-db-init -n db
helm upgrade auth-service ./microservices/auth-service -f ./microservices/auth-service/values-dev.yaml
```

---

## StatefulSets and Persistence

All 4 PostgreSQL instances and Redis use **StatefulSets** with persistent volume claims.

| StatefulSet | Namespace | StorageClass | PVC Size | Headless Service |
|---|---|---|---|---|
| `auth-db` | `db` | `nfs` | `1Gi` | `auth-db-headless` |
| `habit-db` | `db` | `nfs` | `1Gi` | `habit-db-headless` |
| `journal-db` | `db` | `nfs` | `1Gi` | `journal-db-headless` |
| `notification-db` | `db` | `nfs` | `1Gi` | `notification-db-headless` |
| `redis` | `backend` | `nfs` | `1Gi` | `redis-headless` |

The headless service provides stable per-pod DNS in the format:
```
<pod-name>.<headless-svc-name>.<namespace>.svc.cluster.local
```

### Check PVC status

```bash
kubectl get pvc -n db
kubectl get pvc -n backend
# All should be in Bound state
```

### Verify StatefulSet is ready

```bash
kubectl get statefulsets -n db
kubectl get statefulsets -n backend
```

---

## Secrets Management

Each service Deployment uses `envFrom.secretRef` to inject all credentials as environment variables.

### Current Kubernetes Secrets inventory

| Secret Name | Namespace | Keys | Used By |
|---|---|---|---|
| `auth-service-secret` | `backend` | `POSTGRES_URI`, `JWT_SECRET` | auth-service Deployment |
| `habit-service-secret` | `backend` | `POSTGRES_URI`, `JWT_SECRET`, `REDIS_URI` | habit-service Deployment |
| `journal-service-secret` | `backend` | `POSTGRES_URI`, `JWT_SECRET` | journal-service Deployment |
| `notification-service-secret` | `backend` | `POSTGRES_URI`, `REDIS_URI`, `JWT_SECRET` | notification-service + worker Deployments |
| `auth-db-secret` | `db` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | auth-db StatefulSet |
| `habit-db-secret` | `db` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | habit-db StatefulSet |
| `journal-db-secret` | `db` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | journal-db StatefulSet |
| `notification-db-secret` | `db` | `POSTGRES_USER`, `POSTGRES_PASSWORD` | notification-db StatefulSet |
| `redis-secret` | `backend` | `REDIS_PASSWORD` | Redis StatefulSet |

### Verify a secret is present and correct

```bash
# Decode and display a secret value
kubectl get secret auth-service-secret -n backend \
  -o jsonpath='{.data.POSTGRES_URI}' | base64 -d
echo

kubectl get secret auth-service-secret -n backend \
  -o jsonpath='{.data.JWT_SECRET}' | base64 -d
echo
```

> ⚠️ **Known security gap:** Secret values are stored as plaintext in `values-dev.yaml` and `values-prod.yaml`, which are committed to this repository. For a production-hardened setup, migrate to:
> - [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) — encrypts secrets as `SealedSecret` CRDs safe to commit
> - [External Secrets Operator](https://external-secrets.io/) — pulls secrets from Vault, AWS Secrets Manager, etc.

---

## Verification Checklist

Run this after any fresh install or major upgrade:

```bash
# 1. All namespaces exist
kubectl get ns | grep -E "gateway|backend|db|frontend"

# 2. All pods are running
kubectl get pods -A | grep -v Running | grep -v Completed | grep -v Terminating

# 3. All PVCs are bound
kubectl get pvc -A | grep -v Bound

# 4. DB init jobs completed
kubectl get jobs -n db

# 5. Secrets exist in backend namespace
kubectl get secrets -n backend

# 6. Gateway and HTTPRoutes are ready
kubectl get gateway -n gateway
kubectl get httproutes -A

# 7. Network policies are applied
kubectl get networkpolicies -A

# 8. Health check all services via gateway
GATEWAY_IP=<your-gateway-ip>
for svc in auth habits journal notifications; do
  echo -n "$svc: "
  curl -s http://$GATEWAY_IP/$svc/health | jq -r .status
done
```

---

## Upgrading a Service

The normal upgrade path is fully automated via CI/CD:

1. Merge code to `develop` → CI builds `dev-{SHA}` image → CD updates `values-dev.yaml` → ArgoCD syncs dev
2. Create GitHub Release → CI builds semver image → CD updates `values-prod.yaml` → ArgoCD syncs prod

### Manual upgrade (emergency)

```bash
# Update image tag in values file
yq -i '."auth-service".image.tag = "dev-newsha"' microservices/auth-service/values-dev.yaml

# Commit and push (ArgoCD will auto-sync within 3 minutes)
git add microservices/auth-service/values-dev.yaml
git commit -m "cd(auth-service): manual emergency update to dev-newsha"
git push

# OR force-sync immediately via ArgoCD CLI
argocd app sync auth-service
```

### Rollback a service

```bash
# Rollback Helm release
helm rollback auth-service 1 -n backend

# OR via ArgoCD (reverts to last synced git state)
argocd app rollback auth-service
```

---

## Troubleshooting

### Pod stuck in `Pending` — PVC not bound

```bash
kubectl describe pvc <pvc-name> -n db
# Look for: "no persistent volumes available for this claim"
```

Verify the NFS StorageClass provisioner is running:

```bash
kubectl get pods -n nfs-provisioner   # or whichever namespace your provisioner runs in
```

### Pod stuck in `CrashLoopBackOff`

```bash
kubectl logs <pod-name> -n backend --previous
```

Common causes:
- PostgreSQL not yet ready (DB init Job may have failed — check `kubectl get jobs -n db`)
- Wrong `POSTGRES_URI` in the secret — verify with `kubectl get secret ... | base64 -d`
- `JWT_SECRET` mismatch between services

### ArgoCD shows `OutOfSync` but won't sync

```bash
argocd app sync <app-name> --force
argocd app diff <app-name>   # see what ArgoCD wants to change
```

If `selfHeal` is enabled, ArgoCD will keep reverting manual changes. Always commit desired state to this repo first.

### Gateway returns `404` for a route

```bash
kubectl get httproutes -A
kubectl describe httproute <route-name> -n gateway
```

Check that the target Service exists in the correct namespace and the port matches.

### DB init Job fails with `FATAL: role does not exist`

The init Job runs as the PostgreSQL superuser. Verify the `POSTGRES_USER` and `POSTGRES_PASSWORD` in the DB secret match the credentials set in the StatefulSet:

```bash
kubectl get secret auth-db-secret -n db -o jsonpath='{.data.POSTGRES_USER}' | base64 -d
kubectl exec -n db statefulset/auth-db -- psql -U postgres -c "\du"
```

### `helm install` fails with `resource already exists`

A previous install left resources behind. Clean up and retry:

```bash
helm uninstall auth-service -n backend
kubectl delete pvc -n db -l app=auth-db   # Only if you want to wipe data
helm install auth-service ./microservices/auth-service -f ./microservices/auth-service/values-dev.yaml
```
