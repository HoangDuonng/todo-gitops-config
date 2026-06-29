# Todo App GKE Deployment Manifests

This directory contains Kubernetes (K8s) manifests optimized for a 3-node GKE cluster running on `e2-micro` instances.

---

## Repository Structure

The manifests are ordered sequentially from `00` to `10`:

- **00-namespace.yaml**: Defines the `todo-app` Namespace.
- **01-secrets.yaml**: Encrypted credentials (MySQL root password, JWT secret).
- **02-configmaps.yaml**: Network settings, service ports, and gRPC endpoints.
- **03-mysql.yaml**: StatefulSet deploying MySQL with a 5GB PVC and automatic schema initialization.
- **04-redis.yaml**: Deployment deploying Redis for Tyk Gateway caching with a 2GB PVC.
- **05-auth-service.yaml**: Authentication Go microservice.
- **06-user-service.yaml**: User Profile Go microservice.
- **07-task-service.yaml**: Task management Go microservice.
- **08-tyk-gateway.yaml**: Tyk API Gateway configured for proxying `/auth`, `/user`, and `/tasks` endpoints.
- **09-todo-fe.yaml**: Frontend application (React + Vite).
- **10-ingress.yaml**: Ingress routing to expose services externally.

---

## Deployment Instructions

Execute the commands sequentially from the root of this directory:

### Step 1: Initialize Namespace, Secrets, and Configurations

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-secrets.yaml
kubectl apply -f 02-configmaps.yaml
```

### Step 2: Deploy Databases

```bash
kubectl apply -f 03-mysql.yaml
kubectl apply -f 04-redis.yaml
```

_Note: PVC provisioning on GCP may take 1-2 minutes. Verify status before proceeding:_

```bash
kubectl get pods -n todo-app -w
```

_Wait until both database pods show `Running 1/1`._

### Step 3: Deploy Microservices, Gateway, and Frontend

```bash
kubectl apply -f 05-auth-service.yaml
kubectl apply -f 06-user-service.yaml
kubectl apply -f 07-task-service.yaml
kubectl apply -f 08-tyk-gateway.yaml
kubectl apply -f 09-todo-fe.yaml
```

### Step 4: Configure Routing (Ingress)

```bash
kubectl apply -f 10-ingress.yaml
```

---

## Verification & Troubleshooting

### Check Pod Status

```bash
kubectl get pods -n todo-app
```

_Expected output: All pods in `Running` state and `READY` columns as `1/1` (or `2/2` for Tyk Gateway)._

### Retrieve Service Logs

```bash
kubectl logs deployment/auth-service -n todo-app
```

### Get Public IP

```bash
kubectl get ingress todo-ingress -n todo-app
```

_Note: Load balancer provisioning can take 3-5 minutes. Access the application via the IP listed in the `ADDRESS` column (e.g., `http://<EXTERNAL_IP>`)._
