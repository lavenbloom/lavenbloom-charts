# lavenbloom-charts — Study Notes (Part 1 of 2)

> Every file, every attribute, every resource object explained. Written for presenting to a DevOps team.

---

## Table of Contents

1. [Why This Repo Exists — The Real Problem](#1-why-this-repo-exists)
2. [Helm Deep Concepts](#2-helm-deep-concepts)
3. [infrastructure/namespaces](#3-infrastructurenamespaces)
4. [infrastructure/network-policies](#4-infrastructurenetwork-policies)
5. [infrastructure/gateway](#5-infrastructuregateway)
6. [infrastructure/redis](#6-infrastructureredis)
7. [microservices/auth-service — Every File](#7-microservicesauth-service--every-file)
8. [ArgoCD AppProject](#8-argocd-appproject)
9. [ArgoCD ApplicationSets](#9-argocd-applicationsets)
10. [How ArgoCD Works Internally](#10-how-argocd-works-internally)
11. [Environment Management](#11-environment-management)
12. [How a DevOps Engineer Deploys This](#12-how-a-devops-engineer-deploys-this)
13. [How a DevOps Engineer Releases a New Version](#13-how-a-devops-engineer-releases-a-new-version)
14. [Sealed Secrets Deep Dive](#14-sealed-secrets-deep-dive)
15. [StatefulSets vs Deployments](#15-statefulsets-vs-deployments)
16. [Q&A](#16-qa)

---

## 1. Why This Repo Exists

### The real-world problem: configuration drift

Before GitOps, deployments were **imperative**:
```bash
kubectl apply -f deployment.yaml      # someone does this manually
kubectl edit deployment auth-service  # someone patches it live
```
A month later: nobody knows what's actually running. The YAML files are outdated. This is **configuration drift** — the cluster state diverged from your source files.

### This repo is the solution

Everything running in the cluster is declared here. ArgoCD continuously compares this repo against the cluster. Any difference is auto-corrected (`selfHeal: true`). Manual `kubectl` changes are reverted within minutes. The rule is: **if it is not in this repo, it does not exist in the cluster.**

### Why Helm instead of plain YAML?

Plain YAML is static. `auth-service` and `habit-service` need nearly identical Deployment YAML. Helm parameterizes values (`{{ .Values.authService.replicas }}`), separating **structure** (templates) from **configuration** (values files). The CI/CD pipeline only touches the `image.tag` line in `values-dev.yaml` or `values-prod.yaml` — everything else is stable.

### Alternatives to Helm

| Tool | Use case | Notes |
|---|---|---|
| **Helm** ✅ | Parameterized charts, package distribution | Most widely adopted |
| **Kustomize** | Overlay-based patching, no templating | Built into `kubectl` |
| **Jsonnet / cue** | Programmatic YAML generation | Steep learning curve |
| **Pulumi / CDK8s** | Kubernetes config in real languages (Python, TS) | More powerful, more complex |

---

## 2. Helm Deep Concepts

### Chart.yaml — every field

```yaml
# microservices/auth-service/Chart.yaml
apiVersion: v2          # Helm 3 format (v2). v1 = Helm 2 (deprecated)
name: auth-service      # Chart name — must match folder name convention
description: Auth service deployment with PostgreSQL database for the Lavenbloom platform
version: 0.1.0          # Chart version — bump when template structure changes
type: application       # "application" (runnable workload) vs "library" (shared helpers)
```

`apiVersion: v2` means Helm 3 — which dropped the `Tiller` in-cluster server that Helm 2 required. Tiller was a major security risk (it had cluster-admin by default). Helm 3 runs purely client-side.

### values-dev.yaml — every field

```yaml
authService:
  image:
    repository: rnld101/lavenbloom-auth-service
    tag: dev-2192484c87233cf4a58a84c442411c0d366dca7d  # CI writes this
  replicas: 1
  port: 8000
  namespace: backend
  resources:
    requests:
      cpu: 100m       # 0.1 CPU core — guaranteed minimum
      memory: 128Mi   # guaranteed minimum RAM
    limits:
      cpu: 250m       # hard cap — container is throttled if exceeded
      memory: 256Mi   # hard cap — container is OOMKilled if exceeded
db:
  namespace: db
  image: postgres:16-alpine
  replicas: 1
  port: 5432
  storage:
    size: 5Gi
global:
  storageClass: nfs   # Which StorageClass to use for PVCs
```

**CPU millicores:** `100m` = 0.1 core. `1000m` = 1 full core. The scheduler uses `requests` to place pods. `limits` are enforced by the Linux cgroups kernel mechanism.

**`requests` vs `limits`:** Requests = guaranteed slice of node resources. Limits = maximum before throttle/kill. If `requests == limits`, the pod gets a **Guaranteed** QoS class — highest priority, last to be evicted under pressure.

### Template syntax

```yaml
image: "{{ .Values.authService.image.repository }}:{{ .Values.authService.image.tag }}"
```
- `{{ }}` — Go template delimiters
- `.Values` — root of the merged values tree
- `{{- if condition }}` — the `-` trims surrounding whitespace (prevents blank lines in output)
- `{{- range list }}...{{- end }}` — loops over a list

---

## 3. infrastructure/namespaces

### templates/namespaces.yaml

```yaml
{{- range .Values.namespaces }}
---
apiVersion: v1
kind: Namespace
metadata:
  name: {{ . }}
  labels:
    name: {{ . }}   # ← CRITICAL: NetworkPolicy namespaceSelector uses this label
{{- end }}
```

**`range` loop:** Iterates the `namespaces` list from values.yaml, producing one Namespace per item. `{{ . }}` = current loop item.

**Why the `name: {{ . }}` label?** Kubernetes does not automatically label namespaces with their own name. `NetworkPolicy.namespaceSelector.matchLabels` only works if the namespace has the label you're matching on. Without `labels: name: gateway`, the NetworkPolicy rule `matchLabels: name: gateway` would never match — all network isolation would silently break.

### values.yaml

```yaml
namespaces:
  - gateway    # Envoy Gateway controller
  - backend    # Microservice Deployments + Redis
  - db         # PostgreSQL StatefulSets
  - frontend   # React/Nginx Deployment
```

**Why 4 namespaces?**

| Namespace | Reason for separation |
|---|---|
| `gateway` | External entry point — different RBAC, different network rules |
| `backend` | App logic — can be scaled/restarted independently |
| `db` | Data — most sensitive, most restricted network access |
| `frontend` | Static serving — no secrets, different scaling pattern |

---

## 4. infrastructure/network-policies

### templates/network-policies.yaml — all 9 policies

**Policy 1-3: Default deny-all**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-default-deny
  namespace: backend
spec:
  podSelector: {}       # Empty = matches ALL pods in this namespace
  policyTypes:
    - Ingress            # Only restricts incoming traffic
                         # No ingress: rules = deny ALL incoming
```

`podSelector: {}` with no `ingress:` rules = everything is blocked. Three of these exist: `backend-default-deny`, `db-default-deny`, `frontend-default-deny`. This is the **zero-trust baseline**.

**Policy 4: Allow gateway → backend**

```yaml
metadata:
  name: backend-allow-gateway-only
  namespace: backend
spec:
  podSelector:
    matchLabels:
      tier: backend           # Applies to all backend pods
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: gateway   # Traffic MUST come from namespace labeled name=gateway
```

Allows external traffic to reach backend pods — but ONLY if it comes through the gateway namespace. Direct internet access to backend pods is impossible.

**Policy 5: Allow gateway → frontend**

Same pattern as policy 4 but targets `tier: frontend` pods in the `frontend` namespace.

**Policy 6-9: Per-service DB isolation**

```yaml
metadata:
  name: auth-db-allow-auth-service-only
  namespace: db
spec:
  podSelector:
    matchLabels:
      app: auth-db
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: backend
          podSelector:          # ← Same from-item (AND logic, not OR)
            matchLabels:
              app: auth-service
```

**AND vs OR logic:** Both `namespaceSelector` and `podSelector` are under the same `- from:` item without a `-` before `podSelector`. This means: traffic must come from a pod that is BOTH in namespace `backend` AND has label `app: auth-service`. If they were separate `- from:` items, it would be OR — any pod in backend OR any pod labelled auth-service anywhere.

This means `habit-service` (in `backend`) cannot reach `auth-db` even though it's in the same namespace.

**Policy: Redis allow from backend tier**

```yaml
metadata:
  name: redis-allow-backend-services
spec:
  podSelector:
    matchLabels:
      app: redis
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: backend
          podSelector:
            matchLabels:
              tier: backend    # Any pod with tier=backend label can reach Redis
```

Both `habit-service` and `notification-service` have `tier: backend` label, so both can connect to Redis.

### values.yaml

```yaml
gatewayNamespace: gateway
backendNamespace: backend
dbNamespace: db
frontendNamespace: frontend
redisNamespace: backend
```

All namespace names injected via Helm values — rename a namespace in one place and all 9 policies update.

---

## 5. infrastructure/gateway

### templates/gateway-parameters.yaml

```yaml
apiVersion: gateway.kgateway.dev/v1alpha1
kind: GatewayParameters
metadata:
  name: lavenbloom-gw-params
  namespace: gateway
spec:
  kube:
    service:
      type: NodePort
      ports:
        - port: 80
          nodePort: 30080    # Fixed external port on every cluster node
```

`GatewayParameters` is a **kgateway-specific CRD** (not part of the standard Gateway API). It configures the *underlying Kubernetes Service* that the gateway controller creates when a Gateway object is deployed. Without this, the controller defaults to `ClusterIP` — not reachable externally. Setting `NodePort: 30080` means the app is reachable at `<any-node-ip>:30080` without a cloud load balancer — important for bare-metal/on-prem clusters.

The `{{- if .Values.gatewayParameters.enabled }}` wrapping allows disabling this resource for clusters that manage the Service type differently.

### templates/gateway.yaml

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: lavenbloom-gateway
  namespace: gateway
spec:
  gatewayClassName: kgateway    # Controller that manages this Gateway
  infrastructure:
    parametersRef:
      group: gateway.kgateway.dev
      kind: GatewayParameters
      name: lavenbloom-gw-params   # Links to the GatewayParameters above
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All              # HTTPRoutes from ANY namespace can attach here
```

**Gateway API object model:**
- `GatewayClass` — installed by the cluster admin (defines which controller runs)
- `Gateway` — infrastructure team creates this (defines ports/protocols)
- `HTTPRoute` — app team creates this (defines routing rules)

This separation lets platform teams control the gateway config while app teams control their own routes — without needing cluster-admin access.

### templates/httproute-auth.yaml (representative of all 5 routes)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: auth-route
  namespace: backend             # Lives in the backend namespace
spec:
  parentRefs:
    - name: lavenbloom-gateway   # Attaches to this Gateway
      namespace: gateway          # Cross-namespace reference (new in Gateway API)
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /auth         # Match all paths starting with /auth
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /  # Strip /auth → forward / to the service
      backendRefs:
        - name: auth-service
          namespace: backend
          port: 8000
```

**URL rewrite filter:** A request to `/auth/register` becomes `/register` before hitting the FastAPI service. FastAPI's routes are `/register`, `/login` — not `/auth/register`. Without the rewrite, every request would return 404.

**Frontend route has no rewrite filter** — requests to `/` are forwarded as-is because the React app serves from root.

### values.yaml

```yaml
gateway:
  name: lavenbloom-gateway
  namespace: gateway
  gatewayClassName: kgateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
  parametersRef:
    group: gateway.kgateway.dev
    kind: GatewayParameters
    name: lavenbloom-gw-params

gatewayParameters:
  enabled: true
  name: lavenbloom-gw-params
  serviceType: NodePort
  nodePorts:
    - port: 80
      targetPort: 8080
      nodePort: 30080

routes:
  auth:
    parentGatewayNamespace: gateway
    backendService: auth-service
    backendNamespace: backend
    backendPort: 8000
    pathPrefix: /auth
  # ... habit, journal, notification, frontend similarly
```

All route configuration is in values — no hardcoding in templates. Adding a new service requires adding a new block under `routes:` and a new HTTPRoute template.

---

## 6. infrastructure/redis

### templates/statefulset.yaml — every attribute

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: backend
  labels:
    app: redis
spec:
  serviceName: redis-headless   # MUST match the headless Service name below
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine  # Alpine = minimal OS, ~30MB, fewer CVEs
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 6379
          command:
            - redis-server
            - --requirepass
            - $(REDIS_PASSWORD)    # Shell substitution — injects env var as CLI arg
          env:
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: redis-secret      # Read from Kubernetes Secret
                  key: REDIS_PASSWORD
          volumeMounts:
            - name: redis-data
              mountPath: /data           # Redis writes AOF/RDB files here
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: nfs
        resources:
          requests:
            storage: 1Gi
```

**Why `command:` with `$(REDIS_PASSWORD)` instead of `envFrom:`?** Redis doesn't read a `REDIS_PASSWORD` environment variable — it reads the password from its `--requirepass` CLI argument. `$(REDIS_PASSWORD)` is **Kubernetes variable expansion** (not shell) — Kubernetes substitutes the env var value into the command before passing it to the container.

**`volumeClaimTemplates`** is StatefulSet-specific. Creates one PVC per pod: `redis-data-redis-0`. If the pod is deleted and recreated, Kubernetes re-attaches the same PVC. Data persists across pod restarts and node failures.

### templates/service.yaml — two services in one file

**ClusterIP Service:** `redis.backend.svc.cluster.local:6379` — stable virtual IP for connections from `habit-service` and `notification-service`.

**Headless Service (`clusterIP: None`):** `redis-0.redis-headless.backend.svc.cluster.local` — DNS returns the actual pod IP directly. Required by StatefulSet for stable pod identity. Referenced as `serviceName:` in the StatefulSet spec.

---

## 7. microservices/auth-service — Every File

### templates/deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: backend
  labels:
    app: auth-service
    tier: backend          # ← NetworkPolicy selects backend pods by this label
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service    # Deployment owns pods with this label
  template:
    metadata:
      labels:
        app: auth-service
        tier: backend      # ← Pod must have this label for NetworkPolicy to apply
    spec:
      containers:
        - name: auth-service
          image: "rnld101/lavenbloom-auth-service:dev-abc..."
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: auth-service-secret  # Injects POSTGRES_URI + JWT_SECRET as env vars
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 250m
              memory: 256Mi
```

**`selector.matchLabels` must match `template.metadata.labels`** — Kubernetes rejects the Deployment if they don't match. The selector is immutable after creation — to change it you must delete and recreate the Deployment.

**`envFrom.secretRef`** — injects ALL keys from the named Secret as environment variables. Cleaner than listing each key individually with `env.valueFrom.secretKeyRef`.

**Missing in production:** No `livenessProbe` or `readinessProbe`. A liveness probe restarts hung containers. A readiness probe removes non-ready pods from the Service endpoints, preventing traffic from reaching a pod that hasn't finished starting up.

### templates/service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: backend
spec:
  type: ClusterIP         # Internal only — gateway routes external traffic in
  ports:
    - port: 8000          # Service port — what callers connect to
      targetPort: 8000    # Container port — what the app listens on
  selector:
    app: auth-service     # Routes to pods with this label
```

**Why ClusterIP?** External traffic enters via the Gateway's NodePort (30080). The Gateway forwards to this ClusterIP service. The service is never exposed directly outside the cluster.

### templates/sealed-secret.yaml

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: auth-service-secret
  namespace: backend
spec:
  encryptedData:
    JWT_SECRET: AgAg+U5hMePbCm...   # Encrypted with cluster's public key
    POSTGRES_URI: AgC4Rts715JCB...  # Each value encrypted individually
  template:
    metadata:
      name: auth-service-secret
      namespace: backend
    type: Opaque                     # Standard opaque secret type
```

The `template:` section defines what the *decrypted* Secret looks like. The Sealed Secrets controller creates exactly this Secret in the cluster. Encryption is namespace-scoped — this SealedSecret cannot be deployed to a different namespace.

### templates/db/statefulset.yaml

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: auth-db
  namespace: db
spec:
  serviceName: auth-db-headless   # Required: headless service for stable DNS
  replicas: 1
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef:
                name: auth-db-secret   # POSTGRES_USER, POSTGRES_PASSWORD, POSTGRES_DB
          volumeMounts:
            - name: auth-db-data
              mountPath: /var/lib/postgresql/data   # PostgreSQL data directory
              subPath: pgdata    # Use subfolder — avoids NFS lost+found conflict
  volumeClaimTemplates:
    - metadata:
        name: auth-db-data
      spec:
        accessModes: [ReadWriteOnce]
        storageClassName: nfs
        resources:
          requests:
            storage: 5Gi
```

**`subPath: pgdata`** — PostgreSQL requires its data directory to be completely empty on first init. NFS volumes may have a `lost+found` directory at root. `subPath: pgdata` mounts `<nfs-volume>/pgdata/` into the container — an empty subdirectory that PostgreSQL accepts.

**`postgres:16-alpine`** — PostgreSQL 16 on Alpine Linux. Alpine base image is ~5MB vs ~150MB for Debian. Fewer OS packages = smaller attack surface for Trivy scans.

### templates/db/service.yaml — two services

**`auth-db` (ClusterIP):** Used by auth-service to connect via `auth-db.db.svc.cluster.local:5432`.

**`auth-db-headless` (clusterIP: None):** Required by the StatefulSet for stable per-pod DNS. Referenced in `serviceName: auth-db-headless`.

### templates/db/sealed-secret.yaml

Encrypts three PostgreSQL environment variables:
- `POSTGRES_DB` — database name to create on init
- `POSTGRES_USER` — application user to create
- `POSTGRES_PASSWORD` — password for the application user

PostgreSQL's official Docker image reads these variables and initializes the database on first start. All three are encrypted individually — rotating one doesn't require re-sealing the others.

---

## 8. ArgoCD AppProject

### argocd/projects/lavenbloom.yaml — every field

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: lavenbloom
  namespace: argocd
spec:
  description: Lavenbloom project for infrastructure and microservices

  sourceRepos:
    - 'https://github.com/lavenbloom/lavenbloom-charts'
    # Only Applications in this project can pull from this repo
    # If someone tries to add an Application pointing to a different repo, ArgoCD rejects it

  destinations:
    - namespace: '*'                          # Allow deploying to any namespace
      server: https://kubernetes.default.svc  # In-cluster API server

  clusterResourceWhitelist:
    - group: '*'
      kind: '*'   # Allow creating ANY cluster-scoped resource (GatewayClass, ClusterRole, etc.)
```

**What `AppProject` enforces:**
- A team cannot create ArgoCD Applications that pull from unauthorized repos
- A team cannot deploy to namespaces outside their project's `destinations`
- Cluster-level resources (like CRDs) must be explicitly allowed

**`server: https://kubernetes.default.svc`** — ArgoCD is running inside the same cluster it's deploying to. This is the in-cluster API endpoint. For remote clusters, you'd add the external API URL.

**`clusterResourceWhitelist: group: '*', kind: '*'`** — Allows all cluster-scoped resources. Needed because infrastructure charts create resources like `GatewayClass`, `GatewayParameters`, and network policies which are cluster-scoped. In stricter multi-team setups, you'd list only: `[{group: networking.k8s.io, kind: NetworkPolicy}]` etc.

---

## 9. ArgoCD ApplicationSets

### dev/microservices-appset.yaml — every field

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: dev-microservices
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/lavenbloom/lavenbloom-charts
        revision: develop          # Watch the develop branch
        directories:
          - path: microservices/*  # Match every subfolder under microservices/
  template:
    metadata:
      name: '{{path.basename}}'   # App name = folder name: auth-service, habit-service, etc.
    spec:
      project: lavenbloom
      source:
        repoURL: https://github.com/lavenbloom/lavenbloom-charts
        targetRevision: develop
        path: '{{path}}'           # Full path: microservices/auth-service
        helm:
          valueFiles:
            - values-dev.yaml
      destination:
        server: https://kubernetes.default.svc
        # No namespace set — each chart's templates specify their own namespace
      syncPolicy:
        automated:
          prune: true        # Delete cluster resources removed from git
          selfHeal: true     # Revert manual kubectl changes
        syncOptions:
          - CreateNamespace=true  # Create namespace if it doesn't exist
```

**Git directories generator:** Scans the repo for folders matching `microservices/*`. Currently finds 5 folders → creates 5 Applications: `auth-service`, `habit-service`, `journal-service`, `notification-service`, `frontend`. Adding a new folder automatically creates a new Application — no manual registration needed.

**`{{path.basename}}`** — Template variable. If the discovered path is `microservices/auth-service`, `path.basename` = `auth-service`.

**`prune: true`** — If you delete a Service from a Helm chart and push to git, ArgoCD deletes it from the cluster too. Without prune, deleted resources accumulate indefinitely.

**`selfHeal: true`** — Any change made with `kubectl edit` or `kubectl apply` that contradicts git is reverted within the next poll cycle (~3 minutes). This enforces git as the single source of truth.

**`CreateNamespace=true`** — ArgoCD can create the Kubernetes namespace before applying resources. Useful for fresh cluster setups where namespaces don't yet exist.

### dev/infra-appset.yaml vs microservices-appset.yaml

| Attribute | infra-appset | microservices-appset |
|---|---|---|
| `path` scanned | `infrastructure/*` | `microservices/*` |
| `valueFiles` | `values.yaml` | `values-dev.yaml` (dev) / `values-prod.yaml` (prod) |
| Why | Infra is same across envs | Microservices have env-specific image tags |

### prod/microservices-appset.yaml differences

```yaml
metadata:
  name: prod-microservices    # Different name
spec:
  generators:
    - git:
        revision: main        # Watch main branch instead of develop
  template:
    spec:
      source:
        targetRevision: main
        helm:
          valueFiles:
            - values-prod.yaml  # Use prod values
```

Applied to the **prod cluster** only. Watches the `main` branch. The CI/CD pipeline merges `develop` → `main` only when a GitHub Release is created.

---

## 10. How ArgoCD Works Internally

### The control loop

ArgoCD runs 4 controllers inside the cluster:
1. **Application Controller** — manages Application resources, detects drift, triggers syncs
2. **Repo Server** — clones git repos, renders Helm charts, caches results
3. **API Server** — REST/gRPC interface for CLI and Web UI
4. **ApplicationSet Controller** — watches ApplicationSet resources, generates Applications

### Reconciliation cycle (every ~3 minutes)

```
ArgoCD polls git repo at targetRevision
        ↓
Repo Server clones/fetches the branch
        ↓
Runs: helm template ./microservices/auth-service -f values-dev.yaml
        ↓
Produces desired Kubernetes manifests (YAML)
        ↓
Compares against live cluster state via kubectl/API
        ↓
If different (OutOfSync):
  selfHeal=true → immediately applies diff
  selfHeal=false → shows OutOfSync, waits for human
```

### Sync vs Health

**Sync status:** Does the cluster match git?
- `Synced` — cluster matches git exactly
- `OutOfSync` — difference detected (new commit not yet applied, or manual change detected)

**Health status:** Is the deployed app actually working?
- `Healthy` — all pods running, all deployments at desired replicas
- `Progressing` — rollout in progress
- `Degraded` — pods crashing, rollout failed
- `Unknown` — ArgoCD doesn't know how to assess (custom resources)

### ArgoCD vs alternatives

| Tool | Model | UI | Notes |
|---|---|---|---|
| **ArgoCD** ✅ | Pull (cluster polls git) | Rich Web UI | Kubernetes-native GitOps |
| **Flux CD** | Pull | CLI only | Lighter weight, more minimal |
| **Spinnaker** | Push | Very rich | Complex setup, legacy |
| **Jenkins + kubectl** | Push | Jenkins UI | Simple but cluster needs credentials in CI |
| **Helm + GitHub Actions** | Push | None | No drift detection |

**Pull-based (ArgoCD) advantage:** CI never needs cluster credentials. The cluster pulls changes to itself. This reduces the attack surface — a compromised CI pipeline cannot directly deploy to the cluster.

---

## 11. Environment Management

### Two clusters, not two namespaces

Using namespaces for env separation is risky:
- A misconfigured NetworkPolicy could allow dev → prod DB access
- Resource exhaustion in dev starves prod workloads
- A cluster admin mistake in dev affects prod

Two separate clusters give hard boundaries: different API servers, different network segments, different RBAC, different ArgoCD instances.

### Git branch → cluster mapping

```
developer opens PR
    → SAST + SCA + Trivy gate
    
PR merged to develop
    → CI builds image: dev-{SHA}
    → CD updates charts/develop branch: values-dev.yaml
    → ArgoCD DEV cluster detects commit
    → Syncs dev cluster

GitHub Release created (e.g. v1.2.0)
    → CI builds image: v1.2.0
    → CD updates charts/main branch: values-prod.yaml
    → ArgoCD PROD cluster detects commit
    → Syncs prod cluster
```

### Image tag strategy

| Environment | Tag format | Example | Why |
|---|---|---|---|
| Dev | `dev-{full-SHA}` | `dev-2192484c8723...` | Uniquely identifies the commit, full trace |
| Prod | Semver from Release | `2.0.0` | Human-readable, follows industry standard |

---

## 12. How a Real DevOps Engineer Deploys This Application

### Day 0: Bootstrap (done once per cluster)

```bash
# 1. Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 2. Install Sealed Secrets controller (BEFORE applying any SealedSecret resources)
kubectl apply -f https://github.com/bitnami-labs/sealed-secrets/releases/latest/download/controller.yaml

# 3. Install kgateway (Envoy)
helm install kgateway oci://ghcr.io/kgateway-dev/charts/kgateway \
  --namespace kgateway-system --create-namespace

# 4. Create the NFS StorageClass (cluster-specific setup)
kubectl apply -f nfs-storageclass.yaml

# 5. Apply ArgoCD Project (defines RBAC boundaries)
kubectl apply -f argocd/projects/lavenbloom.yaml

# 6. Apply ApplicationSets (ArgoCD starts creating and syncing all Applications)
kubectl apply -f argocd/environments/dev/infra-appset.yaml -n argocd
kubectl apply -f argocd/environments/dev/microservices-appset.yaml -n argocd
```

After step 6, ArgoCD discovers all folders and syncs everything within minutes. No manual `helm install` needed.

### Verification after bootstrap

```bash
# Watch ArgoCD Applications come online
argocd app list
kubectl get pods -n backend
kubectl get pods -n db
kubectl get pvc -A

# Smoke test
curl http://<node-ip>:30080/auth/health
# {"status":"ok"}
```

---

## 13. How a Real DevOps Engineer Releases a New Version

### Step-by-step production release

**1. Merge all feature branches to `develop`**
Each merge triggers dev pipeline → deploys to dev cluster.

**2. QA signs off on dev environment**
Test all APIs, confirm no regressions.

**3. Create a GitHub Release on the service repo**
```
GitHub UI: <service-repo> → Releases → Draft a new release
Tag:       v1.2.0          (semantic versioning: major.minor.patch)
Branch:    main
Title:     Release 1.2.0 — Add journal search feature
Body:      [what changed, what was fixed]
→ Publish Release
```

**4. Automated pipeline takes over**
- `release.types: [created]` event fires
- `ci-docker-publish.yml` builds image tagged `v1.2.0`, pushes to Docker Hub
- `cd-template.yml` updates `values-prod.yaml`: `tag: "v1.2.0"`, commits to `charts/main`

**5. ArgoCD (prod cluster) detects the commit**
Within 3 minutes, ArgoCD runs `helm upgrade` on the changed service.
Kubernetes performs a **rolling update** — new pod starts, old pod terminates.

**6. Monitor the rollout**
```bash
kubectl rollout status deployment/auth-service -n backend
kubectl get pods -n backend -w
```

**7. Post-release verification**
```bash
curl http://<prod-node>:30080/auth/health
# {"status":"ok"}
```

### Rollback procedure

```bash
# Option 1: Git revert (preferred — keeps full audit trail)
# In the charts repo:
git revert <commit-that-updated-values-prod.yaml>
git push origin main
# ArgoCD detects → reverts Deployment image tag → Kubernetes rolls back

# Option 2: Helm rollback (bypasses ArgoCD — use with caution)
helm rollback auth-service 1 -n backend
# NOTE: selfHeal will re-apply the values-prod.yaml tag shortly after
# Pause ArgoCD sync first: argocd app set auth-service --sync-policy none

# Option 3: Force previous image tag via ArgoCD
argocd app set auth-service --helm-set authService.image.tag=v1.1.0
```

---

## 14. Sealed Secrets Deep Dive

### Why plain Kubernetes Secrets are not safe to commit

```bash
echo -n "my-password" | base64
# bXktcGFzc3dvcmQ=
```
Base64 is **encoding, not encryption**. Anyone with the base64 string can decode it in 1 second. Committing plain Secrets to git exposes all credentials to anyone with repo access — including GitHub employees, contractors, or attackers who gain repo access.

### How Sealed Secrets works

```
Developer's machine:
  kubeseal fetches cluster's PUBLIC KEY
  kubeseal encrypts: JWT_SECRET → "AgAg+U5hMePbCm..."
  Developer commits SealedSecret to git (safe — only public key used)

Cluster:
  Sealed Secrets controller holds the PRIVATE KEY (never leaves cluster)
  Controller watches for SealedSecret resources
  Controller decrypts encryptedData using private key
  Controller creates standard Kubernetes Secret with plain values
  Deployment reads Secret via envFrom
```

### Creating/rotating a secret

```bash
# 1. Create the plain secret locally (DO NOT COMMIT THIS)
kubectl create secret generic auth-service-secret \
  --namespace=backend \
  --from-literal=JWT_SECRET=mynewsecretkey \
  --from-literal=POSTGRES_URI="postgresql://user:pass@auth-db.db:5432/auth_db" \
  --dry-run=client -o yaml > /tmp/plain-secret.yaml

# 2. Seal it against the cluster's public key
kubeseal --format yaml < /tmp/plain-secret.yaml \
  > microservices/auth-service/templates/sealed-secret.yaml

# 3. Immediately delete the plain file
rm /tmp/plain-secret.yaml

# 4. Commit and push the sealed file
git add microservices/auth-service/templates/sealed-secret.yaml
git commit -m "secrets: rotate JWT_SECRET for auth-service"
git push
# ArgoCD syncs → controller decrypts → Secret updated in cluster
```

### Alternatives to Sealed Secrets

| Tool | How secrets are managed | Notes |
|---|---|---|
| **Sealed Secrets** ✅ | Encrypted-in-git | Simplest GitOps-compatible approach |
| **External Secrets Operator** | Pull from external vault | Secrets never in git — best for enterprise |
| **HashiCorp Vault** | Dynamic secrets, auto-rotation | Most powerful, most complex |
| **AWS Secrets Manager** | Cloud-native for AWS | Best for EKS |
| **SOPS** | age/PGP encryption of YAML | Good alternative, used with FluxCD |

---

## 15. StatefulSets vs Deployments

### Why databases cannot use Deployments

If PostgreSQL runs as a Deployment:
- Pod gets a random name: `auth-db-7d9f8b-abc`
- Pod is evicted from node A and rescheduled on node B
- New pod gets a **new PVC** (empty — all data lost), OR
- Cannot mount the old PVC because node A still holds it (ReadWriteOnce lock)

**StatefulSet guarantees:**
- Stable name: `auth-db-0`
- Stable DNS: `auth-db-0.auth-db-headless.db.svc.cluster.local`
- Stable PVC: `auth-db-data-auth-db-0` — re-attached to the same pod regardless of which node it runs on

### `ReadWriteOnce` (RWO) explained

A PVC with `accessModes: ReadWriteOnce` can be mounted read-write by **one node at a time**. NFS supports this. If the pod moves to a different node, Kubernetes detaches the PVC from the old node and attaches it to the new node. Data follows the pod.

For multi-replica read replicas you'd need `ReadWriteMany` (NFS supports this) with a primary/replica setup — beyond the current architecture.

### Scaling differences

| Feature | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random hash suffix | Stable ordinal (`redis-0`, `redis-1`) |
| Startup order | All pods start in parallel | `0` then `1` then `2` (ordered) |
| Shutdown order | Reverse of startup | Reverse ordinal order |
| PVC per pod | Shared or separate (manual) | Automatic per-pod PVC via `volumeClaimTemplates` |
| DNS | Service IP only | Per-pod DNS via headless service |

---

## 16. Q&A

**Q: What is the difference between `AppProject` and `ApplicationSet` in ArgoCD?**
A: `AppProject` defines *access boundaries* — which repos, which namespaces, which resource types are allowed. `ApplicationSet` is a generator — it creates multiple `Application` resources from a single template by scanning git directories or a list. They serve different purposes: Project = security boundary, ApplicationSet = automation.

**Q: What happens if the Sealed Secrets controller is deleted?**
A: The private key is stored as a regular Kubernetes Secret in `kube-system`. If the controller pod is deleted, it restarts and continues working. If the `kube-system` Secret containing the private key is deleted, all SealedSecrets become permanently unrecoverable. The private key must be backed up. Command: `kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > sealed-secrets-key-backup.yaml`.

**Q: Why does `selfHeal: true` make sense if developers sometimes need to apply hotfixes directly?**
A: In emergencies, you would either: (1) commit directly to the git branch so ArgoCD applies it — fastest, auditable, or (2) pause ArgoCD sync for that application (`argocd app set auth-service --sync-policy none`), apply the hotfix, then resume sync and commit the proper fix to git. Never leave ArgoCD paused longer than necessary.

**Q: What is the `path.basename` template variable?**
A: `path` is the full directory path discovered by the Git generator (e.g., `microservices/auth-service`). `path.basename` extracts the last segment: `auth-service`. This becomes the Application name in ArgoCD.

**Q: Why is `CreateNamespace=true` set if namespaces are also created by the infra chart?**
A: Defense in depth. The infra ApplicationSet syncs the namespaces chart first, but there's no guaranteed ordering between infra and microservices ApplicationSets. `CreateNamespace=true` ensures the microservices ApplicationSet can deploy even if namespaces aren't pre-created yet — ArgoCD creates them automatically.

**Q: How would you add a new microservice to this platform?**
A: 1. Create `microservices/new-service/` with Chart.yaml + templates/ + values-dev.yaml + values-prod.yaml. 2. Create SealedSecrets (run kubeseal locally). 3. Add HTTPRoute template in `infrastructure/gateway/templates/`. 4. Add NetworkPolicy rules in network-policies.yaml. 5. Commit and push to `develop`. The ApplicationSet's Git generator detects the new folder → creates Application → syncs automatically.

**Q: What does the `---` separator in a YAML file do?**
A: It separates multiple Kubernetes resources within one file. `kubectl apply -f network-policies.yaml` applies all 9 resources in one command. Helm processes each `---`-separated document as a separate resource. This keeps related resources co-located without needing separate files.

**Q: How does ArgoCD handle a failed Helm release?**
A: If `helm upgrade` fails (e.g., a container crashes immediately after start), ArgoCD marks the Application as `Degraded`. It does NOT automatically rollback (that would create a loop). A human must investigate the pod logs, fix the issue in git, and push — ArgoCD then syncs the fix.

**Q: Why `imagePullPolicy: IfNotPresent` instead of `Always`?**
A: Since every image tag is unique (SHA for dev, semver for prod), the same tag will never point to a different image. `IfNotPresent` uses the locally cached image — saves registry bandwidth and speeds up pod startup. `Always` is only needed with mutable tags like `latest`.

**Q: What is the real-world business risk of NOT having NetworkPolicies?**
A: If `habit-service` is compromised (e.g., via a dependency vulnerability), an attacker could connect directly to `auth-db` and read all user credentials. With NetworkPolicies, even a fully compromised backend pod cannot reach databases it's not supposed to. This contains the blast radius of any single component compromise.
