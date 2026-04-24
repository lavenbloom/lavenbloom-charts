# Lavenbloom Helm Charts Repository

A comprehensive Helm charts repository for deploying the Lavenbloom microservices platform on Kubernetes.

## Architecture

The platform consists of four microservices, each with its own PostgreSQL database, a shared Redis instance, and an Envoy Gateway for traffic routing.

### Namespaces

| Namespace   | Purpose                              |
|-------------|--------------------------------------|
| `gateway`   | Envoy Gateway pods                   |
| `backend`   | All microservice deployments + Redis |
| `db`        | All PostgreSQL StatefulSets          |
| `frontend`  | Frontend nginx deployment            |

### Services

| Service               | Env Vars                              | Database         | Port |
|-----------------------|---------------------------------------|------------------|------|
| auth-service          | `POSTGRES_URI`, `JWT_SECRET`          | authdb           | 8000 |
| habit-service         | `POSTGRES_URI`, `JWT_SECRET`, `REDIS_URI` | habitdb      | 8000 |
| journal-service       | `POSTGRES_URI`, `JWT_SECRET`          | journaldb        | 8000 |
| notification-service  | `POSTGRES_URI`, `REDIS_URI`           | notificationdb   | 8000 |
| frontend              | None                                  | None             | 80   |

## Repository Structure

```
charts-repo/
├── infrastructure/          # Cluster-level resources
│   ├── namespaces/          # Namespace definitions
│   ├── gateway/             # Envoy Gateway + HTTPRoutes
│   ├── network-policies/    # NetworkPolicy enforcement
│   └── redis/               # Redis StatefulSet
├── microservices/           # Application charts
│   ├── auth-service/        # Auth API + DB
│   ├── habit-service/       # Habit API + DB
│   ├── journal-service/     # Journal API + DB
│   ├── notification-service/# Notification API + Worker + DB
│   └── frontend/            # Nginx frontend
├── values-dev.yaml          # Dev environment overrides
├── values-prod.yaml         # Prod environment overrides
└── README.md
```

## Deployment

### Prerequisites

- Kubernetes 1.27+
- Helm 3.12+
- Envoy Gateway controller installed (provides the `eg` GatewayClass)

### Install Infrastructure

```bash
# Create namespaces first
helm install namespaces ./infrastructure/namespaces

# Deploy Redis
helm install redis ./infrastructure/redis -f values-dev.yaml

# Deploy Gateway and routes
helm install gateway ./infrastructure/gateway -f values-dev.yaml

# Apply network policies
helm install network-policies ./infrastructure/network-policies -f values-dev.yaml
```

### Install Microservices

```bash
# Deploy each service (dev environment)
helm install auth-service ./microservices/auth-service -f values-dev.yaml
helm install habit-service ./microservices/habit-service -f values-dev.yaml
helm install journal-service ./microservices/journal-service -f values-dev.yaml
helm install notification-service ./microservices/notification-service -f values-dev.yaml
helm install frontend ./microservices/frontend -f values-dev.yaml
```

### Production Deployment

Replace `-f values-dev.yaml` with `-f values-prod.yaml` for production:

```bash
helm install auth-service ./microservices/auth-service -f values-prod.yaml
```

## Network Policy Rules

- **Backend services**: Only accept ingress from the `gateway` namespace
- **Frontend**: Only accepts ingress from the `gateway` namespace
- **Databases**: Each DB only accepts ingress from its corresponding backend service
- **Redis**: Only accepts ingress from `habit-service`
- **No lateral traffic**: Microservices cannot communicate directly with each other

## Gateway Routes

| Path             | Target Service       | Port |
|------------------|---------------------|------|
| `/auth`          | auth-service        | 8000 |
| `/habits`        | habit-service       | 8000 |
| `/journal`       | journal-service     | 8000 |
| `/notifications` | notification-service| 8000 |
| `/`              | frontend            | 80   |

## Database Init Jobs

Each service includes a Helm post-install hook job that:
1. Waits for PostgreSQL to be ready using `pg_isready`
2. Creates the application database
3. Creates the application user with password
4. Grants all privileges on the database to the user

Jobs are idempotent (using `|| true`) and auto-cleanup on success via `hook-delete-policy: hook-succeeded`.
# lavenbloom-charts
