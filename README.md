# 🚀 Microservices Platform — Kubernetes-Ready Backend

This repository contains a fully containerized **microservices backend platform** built with:

- **NestJS** (Auth Service + Post Service)
- **PostgreSQL** (Primary relational database)
- **Redis** (Caching + token/session store)
- **Kong API Gateway** (Routing, policies, rate limiting)
- **Docker Compose** (Local development)
- **Kubernetes / Minikube** (Production-like deployment)

The goal is to provide a realistic, production-style architecture suitable for on‑prem or cloud (EKS / AKS / GKE) deployments.

---

## 🏗️ High-Level Architecture

### Microservices Overview

| Service          | Responsibility                                | Ports (Container) |
|------------------|-----------------------------------------------|-------------------|
| **Auth Service** | Login/Signup, JWT, gRPC server                | 9001 / 50051      |
| **Post Service** | CRUD posts, gRPC client to Auth               | 9002 / 50052      |
| **PostgreSQL**   | Main relational database                      | 5432              |
| **Redis**        | Cache, token store, sessions                  | 6379              |
| **Kong Gateway** | API routing, policies, rate limiting, ingress | 8000 / 30080      |

### Architecture Diagram

```text
             ┌───────────────────────┐
             │      Kong Gateway     │
             │ (Routing / Ratelimit) │
             └─────────┬─────────────┘
                       │
      ┌────────────────┼─────────────────┐
      │                │                 │
  ┌──────────┐    ┌──────────┐     ┌──────────┐
  │  /auth   │    │  /post   │     │  future  │
  │ Auth Svc │    │ Post Svc │     │ services │
  └────┬─────┘    └────┬─────┘     └─────────┘
       │               │
       │ REST + gRPC   │ REST + gRPC
       │               │
   ┌───▼──────┐     ┌───▼─────┐
   │PostgreSQL│     │  Redis  │
   └──────────┘     └─────────┘
```

- External clients talk only to **Kong**.
- Kong forwards traffic to internal services over **ClusterIP**.
- Services use **PostgreSQL** and **Redis** via Kubernetes DNS.

---

## 🐳 Phase 1 — Docker Compose (Local Dev)

Initially, the system runs using **Docker Compose** for fast, developer‑friendly iteration:

- Each service has its own build context, ports, environment variables, volumes, and healthchecks.
- Networking is handled via a local Docker network, allowing containers to reach each other by name.

### Why Docker Compose?

- Simple local setup (`docker-compose up --build`).
- Easy to rebuild, restart, and attach logs.
- Great for single‑node, non‑production development.

### Limitations Encountered

| Limitation          | Impact in Practice                                      |
|---------------------|---------------------------------------------------------|
| No self‑healing     | Crashed containers stay down until restarted manually   |
| No auto‑scaling     | Cannot scale services horizontally under load          |
| No rolling updates  | Deployments cause downtime                             |
| Manual networking   | Service URLs must be maintained manually               |
| No service discovery| Tight coupling to container names and network configs  |

To address these production gaps, the platform is migrated to **Kubernetes**.

---

## ☸️ Phase 2 — Why Kubernetes?

Kubernetes provides the building blocks needed for a production‑ready microservices platform:

| Kubernetes Feature | How it Helps This System                                          |
|--------------------|-------------------------------------------------------------------|
| **Deployments**    | Self‑healing pods, restarts on failure, rolling updates          |
| **Services**       | Stable virtual IPs and DNS for service discovery                 |
| **StatefulSets**   | Stable identities and storage for PostgreSQL                     |
| **PVCs**           | Persistent volumes for database data                             |
| **Secrets/ConfigMaps** | Secure and centralized configuration management           |
| **Horizontal Scaling** | Easily scale stateless services like Post Service         |
| **Ingress / Gateway**  | Expose a single entrypoint via Kong as API gateway        |

---

## 🗄️ PostgreSQL — StatefulSet

**Files:**

- `postgres/postgres-secret.yaml`
- `postgres/postgres-pvc.yaml`
- `postgres/postgres-statefulset.yaml`
- `postgres/postgres-service.yaml`

**Key Decisions:**

- Use a **StatefulSet** so each database pod has a stable hostname and persistent identity.
- Attach a **PersistentVolumeClaim (PVC)** to preserve data across pod restarts.
- Store credentials (user, password, database name) in a **Secret**, not in plaintext.
- Expose PostgreSQL via a **ClusterIP Service** named `postgres`, so other services connect using:

```
postgresql://admin:master123@postgres:5432/postgres
```

---

## ⚡ Redis — Deployment + ClusterIP

**Files:**

- `redis/redis-configmap.yaml`
- `redis/redis-deployment.yaml`
- `redis/redis-service.yaml`

**Key Decisions:**

- Single replica (sufficient for dev/demo) via a standard Deployment.
- Exposed as a **ClusterIP Service** named `redis`, reachable as `redis:6379`.
- Configuration (max memory, eviction, etc.) injected via **ConfigMap** for easy tuning.

Redis is used for caching and session/token storage, offloading frequent reads from PostgreSQL.

---

## 🔐 Auth Service — NestJS

**Files:**

- `auth/auth-configmap.yaml`
- `auth/auth-secret.yaml`
- `auth/auth-deployment.yaml`
- `auth/auth-service.yaml`

**Key Decisions:**

- All `.env` values are split into:
  - **ConfigMap** → non‑sensitive configuration.
  - **Secret** → JWT keys, database credentials, etc.
- Database connection updated to use Kubernetes DNS:

```
postgresql://admin:master123@postgres:5432/postgres
```

- Container startup runs Prisma migrations before starting the app:

```bash
yarn prisma migrate deploy
yarn start
```

- Exposed internally as a **ClusterIP Service** (e.g. `auth-service`), with:
  - REST on port `9001`.
  - gRPC server on port `50051`.

---

## 📝 Post Service — NestJS

**Files:**

- `post/post-config.yaml`
- `post/post-secret.yaml`
- `post/post-deployment.yaml`
- `post/post-service.yaml`

**Key Decisions:**

- Same env‑split pattern using ConfigMap + Secret.
- Uses PostgreSQL and Redis similar to the Auth service.
- Requires an extra environment variable for gRPC communication:

```
GRPC_AUTH_URL=auth-service:50051
```

- Resolves `auth-service` using cluster DNS, enabling secure internal‑only gRPC calls.
- Runs Prisma migrations on startup to keep the schema in sync with the database.

---

## 🌉 Kong API Gateway — Ingress Layer

**Files:**

- `api-gateway/kong-configmap.yaml`
- `api-gateway/kong-deployment.yaml`
- `api-gateway/kong-service.yaml`

Kong acts as the single entrypoint into the cluster and handles routing, rate limiting, and service discovery.

### Routes

- `/auth` → forwards to `auth-service:9001`
- `/post` → forwards to `post-service:9002`

### Rate Limiting Plugins

- **Auth routes:**
  - minute: `100`
  - hour: `1000`
- **Post routes:**
  - minute: `200`
  - hour: `2000`

### Service Discovery

Within Kubernetes, Kong reaches services using fully qualified DNS names:

- `auth-service.default.svc.cluster.local`
- `post-service.default.svc.cluster.local`

---

## 🚀 Quick Start

### Local Development with Docker Compose

```bash
# Clone the repository
git clone <your-repo-url>
cd <repo-name>

# Build and start all services
docker-compose up --build

# Run database migrations
docker-compose exec auth-service yarn prisma migrate deploy
docker-compose exec post-service yarn prisma migrate deploy

# Test the system
curl http://localhost:8000/auth
curl http://localhost:8000/post
```

### Production Deployment with Kubernetes

```bash
# Create namespace
kubectl create namespace microservices

# Deploy PostgreSQL
kubectl apply -f postgres/

# Deploy Redis
kubectl apply -f redis/

# Deploy Auth Service
kubectl apply -f auth/

# Deploy Post Service
kubectl apply -f post/

# Deploy Kong Gateway
kubectl apply -f api-gateway/

# Verify all pods are running
kubectl get pods -n default

# Expose Kong and test
kubectl port-forward svc/kong 8000:8000

# Test endpoints
curl http://localhost:8000/auth
curl http://localhost:8000/post
```

---

## 🧪 End-to-End Testing

After deploying all manifests:

1. Get Kong NodePort:

```bash
kubectl get svc kong
```

2. Test Auth Service:

```bash
curl http://<node-ip>:30080/auth
```

3. Test Post Service:

```bash
curl http://<node-ip>:30080/post
```

Expected response (example):

```json
{
  "status": "ok",
  "environment": "local"
}
```

If both endpoints return successfully:

- Kong routing works.
- Services are reachable through the gateway.
- gRPC communication between Post and Auth is healthy.
- PostgreSQL and Redis connections are established.

---

## 📊 Monitoring & Logging

### Docker Compose

```bash
# View logs from a specific service
docker-compose logs -f auth-service

# View all logs
docker-compose logs -f
```

### Kubernetes

```bash
# View logs from a pod
kubectl logs -f deployment/auth-service

# Stream logs from all pods in a service
kubectl logs -f -l app=auth-service

# Describe pod for debugging
kubectl describe pod <pod-name>

# Access pod shell for debugging
kubectl exec -it <pod-name> -- /bin/bash
```

---

## 🔒 Security Best Practices

- All sensitive data (DB passwords, JWT keys) stored in **Kubernetes Secrets**.
- Kong rate limiting protects against abuse.
- gRPC between services is **internal-only** via ClusterIP.
- PostgreSQL exposed only to internal services, not externally.
- Use **network policies** to restrict traffic between pods (not shown in basic setup).

---

## 🛠️ Troubleshooting

### Services can't reach database

- Check PostgreSQL pod is running: `kubectl get pods | grep postgres`
- Verify connection string: `postgresql://admin:master123@postgres:5432/postgres`
- Check logs: `kubectl logs deployment/postgres`

### Kong can't route to services

- Verify service DNS: `kubectl exec -it <kong-pod> -- nslookup auth-service`
- Check Kong configuration in `kong-configmap.yaml`
- Verify Auth/Post services are running: `kubectl get svc`

### Migrations fail

- Check if PostgreSQL is fully initialized before running migrations
- Verify Prisma schema matches database state
- Check migration files exist in the service container

---

## ✅ Final Outcome

This project demonstrates a complete migration from Docker Compose to a Kubernetes-based microservices platform with:

- Stateful storage for PostgreSQL
- Internal DNS-based service discovery
- Automated database migrations on startup
- Secure Secrets and flexible ConfigMaps
- Kong as an API gateway with routing and policies
- A scalable architecture ready for:
  - On‑prem clusters
  - EKS / AKS / GKE
  - CI/CD integration
  - Auto‑scaling, monitoring, and logging

---

## 📦 Project Structure

```
.
├── docker-compose.yml
├── README.md
├── postgres/
│   ├── postgres-secret.yaml
│   ├── postgres-pvc.yaml
│   ├── postgres-statefulset.yaml
│   └── postgres-service.yaml
├── redis/
│   ├── redis-configmap.yaml
│   ├── redis-deployment.yaml
│   └── redis-service.yaml
├── auth/
│   ├── auth-configmap.yaml
│   ├── auth-secret.yaml
│   ├── auth-deployment.yaml
│   └── auth-service.yaml
├── post/
│   ├── post-configmap.yaml
│   ├── post-secret.yaml
│   ├── post-deployment.yaml
│   └── post-service.yaml
└── api-gateway/
    ├── kong-configmap.yaml
    ├── kong-deployment.yaml
    └── kong-service.yaml
```

---
<img width="1905" height="888" alt="image" src="https://github.com/user-attachments/assets/df09ce8f-5906-46ef-befb-6a742ddffef9" />

<img width="1904" height="851" alt="image" src="https://github.com/user-attachments/assets/e18fc988-3f18-4551-ad24-180a4c7b70c9" />

---
<img width="1224" height="826" alt="image" src="https://github.com/user-attachments/assets/6e3526a0-7cd6-41f0-9de7-219f7b8f031b" />

## 👤 Author

**Abdelrahman Ahmed**  
_2025 — Microservices Migration Project_

For questions or contributions, please open an issue or submit a pull request.
