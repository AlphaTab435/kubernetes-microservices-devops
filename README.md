# Kubernetes Microservices Deployment Project

This repository contains the Kubernetes manifests and deployment workflow for a multi-tier microservices application.

> **Note:** This infrastructure was deployed, tested, and validated within a **Killercoda Kubernetes Sandbox** environment. Because sandboxes lack external cloud-provider integrations, LoadBalancer IPs remain pending, and external traffic routing is achieved via standard Gateway API / Service port-forwarding rather than external DNS.

- **Source Reference:** [saiyam1814/Kubernetes-crash-course-2025](https://github.com/saiyam1814/Kubernetes-crash-course-2025)
- **Main Repository:** [AlphaTab435/kubernetes-microservices-devops](https://github.com/AlphaTab435/kubernetes-microservices-devops)

---

## 📐 Architecture Overview

- **Database Tier:** [CloudNative-PG](https://cloudnative-pg.io/) (CNPG) High-Availability PostgreSQL 3-node cluster with `init.sql` schema bootstrap.
- **Application Tier:**
  - `auth-service` (User Authentication)
  - `game-service` (Game Logic & Leaderboards)
  - `frontend` (Nginx Reverse Proxy & UI)
- **Ingress & Gateway Tier:** [KGateway](https://kgateway.dev/) implementing **Kubernetes Gateway API v1.2.1** (*No-DNS / Direct IP Setup*).
- **Cert & Traffic Management:** `cert-manager` with Gateway API feature flag enabled.

---

## 📹 Demo & Video Walkthrough

Below is the visual preview of the application deployment and port-forwarding testing without a DNS setup:

<!-- Drag and drop your .mp4 or .gif file right here in the GitHub editor to auto-generate the visual. -->
https://github.com/user-attachments/assets/f5183d76-51d9-49fa-b784-e28903c2e03b

---

## 🚀 Quickstart & Deployment Guide

### Step 1: Clone the Repository

```bash
git clone https://github.com/AlphaTab435/kubernetes-microservices-devops
cd kubernetes-microservices-devops
```

---

### Step 2: Install CloudNative-PG Operator & Prepare Database Namespace

1. **Install CNPG Operator (v1.25.2):**

   ```bash
   kubectl apply --server-side -f \
     https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.2.yaml
   ```

2. **Create Target Namespace (`crash-course`):**

   ```bash
   kubectl create ns crash-course
   ```

3. **Create Database Initialization ConfigMap:**

   ```bash
   kubectl create configmap init-sql \
     --from-file=init.sql=./init.sql \
     -n crash-course
   ```

4. **Deploy CNPG PostgreSQL Cluster:**

   ```bash
   kubectl apply -f manifests/pgcluster.yaml
   ```

5. **Verify Database Health:**

   ```bash
   kubectl get pods,svc,jobs -n crash-course
   ```

   *Wait until `postgres-1`, `postgres-2`, and `postgres-3` pods are in a `1/1 Running` state.*

---

### Step 3: Deploy Microservices & Services

> ⚠️ **Important Sequence Note:** Apply both Deployment and Service manifests together so the Nginx container in `frontend` can successfully resolve `auth-service.crash-course.svc.cluster.local` upon startup.

1. **Deploy Microservice Applications:**

   ```bash
   kubectl apply -f manifests/auth-deploy.yaml
   kubectl apply -f manifests/game-deploy.yaml
   kubectl apply -f manifests/frontend.yaml
   ```

2. **Expose Microservices via ClusterIP Services:**

   ```bash
   kubectl apply -f manifests/auth-service.yaml
   kubectl apply -f manifests/game-service.yaml
   kubectl apply -f manifests/frontend-service.yaml
   ```

3. **Restart Frontend Deployment (if Nginx threw host lookup errors prior to service creation):**

   ```bash
   kubectl rollout restart deployment/frontend -n crash-course
   ```

4. **Verify Application Status:**

   ```bash
   kubectl get all -n crash-course
   ```

---

### Step 4: Quick Port-Forwarding Test

To quickly verify application responsiveness before Gateway setup (as tested in the sandbox):

```bash
kubectl port-forward svc/frontend 8080:80 -n crash-course
```

Access the application locally via `http://localhost:8080`.

---

### Step 5: Gateway API & KGateway Deployment (No-DNS Setup)

This step configures traffic routing using the Kubernetes Gateway API without requiring external DNS configuration.

#### 1. Install & Configure Cert-Manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.0/cert-manager.yaml
```

Enable the `--enable-gateway-api` flag in the cert-manager deployment:

```bash
kubectl edit deploy cert-manager -n cert-manager
```

*Add `--enable-gateway-api` under the container `args`, then restart:*

```bash
kubectl rollout restart deployment cert-manager -n cert-manager
```

#### 2. Install Gateway API CRDs (v1.2.1 Standard)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

#### 3. Install KGateway via Helm

```bash
helm upgrade -i --create-namespace --namespace kgateway-system --version v2.0.1 kgateway-crds oci://cr.kgateway.dev/kgateway-dev/charts/kgateway-crds
helm upgrade -i --namespace kgateway-system --version v2.0.1 kgateway oci://cr.kgateway.dev/kgateway-dev/charts/kgateway
```

#### 4. Apply Gateway & HTTPRoute (No-Domain Manifests)

```bash
kubectl apply -f manifests/gateway-no-domain.yaml
kubectl apply -f manifests/httproute-no-domain.yaml
```

#### 5. Verify Gateway Control Plane

```bash
kubectl get all -n kgateway-system
```

---

## 🛠️ Troubleshooting Checklist

| Issue | Cause | Resolution |
| --- | --- | --- |
| `namespaces "crash-course" not found` | Attempted creating ConfigMap before namespace creation. | Run `kubectl create ns crash-course` before creating ConfigMaps. |
| `[emerg] host not found in upstream "auth-service..."` | Frontend container started before K8s Service object was created. | Apply `auth-service.yaml` and run `kubectl rollout restart deployment/frontend -n crash-course`. |
| `LoadBalancer IP <pending>` | Sandbox environment lacks an external cloud provider IP provisioner. | Use NodePort / IP routing provided by `gateway-no-domain.yaml` or basic port-forwarding. |

---

## 📂 Repository Manifest Directory Structure

```text
manifests/
├── auth-deploy.yaml          # Auth Service Deployment
├── auth-service.yaml         # Auth Service ClusterIP
├── cluster-issuer.yaml       # Cert-Manager Issuer Configuration
├── configmap.yaml            # General ConfigMaps
├── frontend.yaml             # Frontend Nginx Deployment
├── frontend-service.yaml     # Frontend Service ClusterIP
├── game-deploy.yaml          # Game Service Deployment
├── game-service.yaml         # Game Service ClusterIP
├── gateway.yaml              # Domain Gateway API Configuration
├── gateway-no-domain.yaml    # Direct IP Gateway API (No-DNS)
├── httpredirect.yaml         # HTTP Redirect Rules
├── httproute.yaml            # Domain HTTPRoute Configuration
├── httproute-no-domain.yaml  # Direct IP HTTPRoute (No-DNS)
├── pgcluster.yaml            # CNPG PostgreSQL HA Cluster Spec
└── servicemonitor.yaml       # Prometheus Monitoring Spec
```
