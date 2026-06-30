# Todo App GKE GitOps & Helm Chart

This repository is structured as a Helm Chart and ArgoCD configuration optimized for deploying the Todo application on a 3-node GKE cluster running on `e2-micro` instances.

---

## 📁 Repository Structure

The root directory contains the Helm Chart and ArgoCD configuration:

- **Chart.yaml**: Chart metadata defining the deployment.
- **values.yaml**: Centralized configuration variables (Git-ignored locally to prevent leaking secrets/custom configs).
- **values-tags.yaml**: Dynamic image tags managed and updated automatically by GitHub Actions CI.
- **argocd-app.yaml**: ArgoCD Application manifest.
- **templates/**: Parameterized Kubernetes resource templates.

---

## 🚀 Bootstrap & Installation Guide

To bootstrap ArgoCD and deploy the application, run the following commands sequentially on your cluster:

### Step 1: Install ArgoCD
```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD (Sử dụng --server-side và --force-conflicts để tránh lỗi dung lượng annotations và xung đột quyền sở hữu)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts
```

### Step 2: Deploy the ArgoCD Application (Bootstrap)
We render the templated Application manifest and apply it to GKE:
```bash
helm template . -s templates/argocd-app.yaml | kubectl apply -f -
```

---

## 🔍 Verification & Troubleshooting

### Check Rollout Status
```bash
kubectl get pods -n todo-app
```
*Expected output: All pods in `Running` state and `READY` columns as `1/1` (or `2/2` for Tyk Gateway).*

### Retrieve Service Logs
```bash
kubectl logs deployment/auth-service -n todo-app
```
