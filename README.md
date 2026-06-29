# Todo App GKE Deployment Chart

This repository is structured as a Helm Chart optimized for deploying the Todo application on a 3-node GKE cluster running on `e2-micro` instances.

---

## Repository Structure

The root directory contains the Helm Chart structure:

- **Chart.yaml**: Chart metadata defining the deployment.
- **values.yaml**: Centralized configuration variables (Git-ignored locally to prevent leaking secrets/custom configs).
- **templates/**: Parameterized Kubernetes resource templates:
  - **namespace.yaml**: Defines the `todo-app` Namespace.
  - **secrets.yaml**: Decrypts and encodes sensitive credentials.
  - **configmaps.yaml**: Centralized network ports and endpoints.
  - **mysql.yaml**: StatefulSet deploying MySQL with 5GB PVC.
  - **redis.yaml**: Deployment deploying Redis for Tyk Gateway caching with 2GB PVC.
  - **auth-service.yaml**: Authentication Go microservice.
  - **user-service.yaml**: User Profile Go microservice.
  - **task-service.yaml**: Task management Go microservice.
  - **tyk-gateway.yaml**: Tyk API Gateway proxying public traffic.
  - **todo-fe.yaml**: Frontend application (React + Vite).
  - **ingress.yaml**: Ingress routing rules for public traffic.

---

## Deployment Instructions

Ensure your Kubernetes context is set to your GKE cluster, then execute the following command from the root of this repository:

### Install / Upgrade the Chart

```bash
helm upgrade --install todo-app . -n todo-app --create-namespace
```

---

## Verification & Troubleshooting

### Check Rollout Status

```bash
kubectl get pods -n todo-app
```

_Expected output: All pods in `Running` state and `READY` columns as `1/1` (or `2/2` for Tyk Gateway)._

### Retrieve Service Logs

```bash
kubectl logs deployment/auth-service -n todo-app
```

### Accessing the Application

If Ingress is configured:

```bash
kubectl get ingress todo-ingress -n todo-app
```

_Access the external IP listed under the `ADDRESS` column._

For local development port-forwarding:

```bash
kubectl port-forward svc/todo-fe 8080:80 -n todo-app
```

_Open [http://127.0.0.1:8080](http://127.0.0.1:8080) in your browser._
