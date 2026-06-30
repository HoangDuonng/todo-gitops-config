# Todo App GKE GitOps & Helm Chart

This repository is structured as a Helm Chart and ArgoCD configuration optimized for deploying the Todo application on a GKE cluster.

Secrets (passwords, tokens) are managed using the **External Secrets Operator (ESO)** and **Google Cloud Secret Manager (GCP SM)** to ensure they are never committed to Git in plain text.

---

## 📁 Repository Structure

The root directory contains the Helm Chart and ArgoCD configuration:

- **Chart.yaml**: Chart metadata defining the deployment.
- **values.yaml**: Centralized configuration variables (tracked in Git, contains no sensitive data).
- **values-tags.yaml**: Dynamic image tags managed and updated automatically by GitHub Actions CI.
- **templates/**: Parameterized Kubernetes resource templates.
  - **templates/argocd-app.yaml**: ArgoCD Application manifest.
  - **templates/argocd-repo-secret.yaml**: Credentials to access the private GitOps repository.
  - **templates/secret-store.yaml**: Connects Kubernetes to Google Cloud Secret Manager.
  - **templates/external-secret.yaml**: Retrieves passwords from Google Cloud and provisions the K8s `app-secrets` Secret.

---

## 🚀 Bootstrap & Installation Guide

Run the following commands sequentially to set up ArgoCD, External Secrets Operator, GCP Service Accounts, and deploy the application.

### Step 1: Install ArgoCD & External Secrets Operator (ESO)
```bash
# 1. Install ArgoCD
kubectl create namespace argocd || true
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side --force-conflicts

# 2. Install External Secrets Operator (ESO)
helm repo add external-secrets https://charts.external-secrets.io
helm repo update
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true
```

### Step 2: Configure Google Cloud Secret Manager
Create the secrets in your GCP project (`todo-devops-500719`):
```bash
# 1. Create the MySQL Root Password secret
gcloud secrets create mysql-root-password --replication-policy="automatic"
echo -n "root_password_secret_tcp" | gcloud secrets versions add mysql-root-password --data-file=-

# 2. Create the JWT Secret secret
gcloud secrets create jwt-secret --replication-policy="automatic"
echo -n "very-important-please-change-it!" | gcloud secrets versions add jwt-secret --data-file=-
```

### Step 3: Configure Workload Identity for ESO
Allow the External Secrets Operator pod to read GCP Secret Manager using Workload Identity:
```bash
PROJECT_ID="todo-devops-500719"

# 1. Create GCP Service Account (GSA)
gcloud iam service-accounts create external-secrets-sa \
    --display-name="External Secrets Operator GSA" || true

# 2. Grant Secret Accessor role to the GSA
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
    --member="serviceAccount:external-secrets-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
    --role="roles/secretmanager.secretAccessor"

# 3. Bind K8s Service Account (KSA) to GCP GSA
gcloud iam service-accounts add-iam-policy-binding \
    external-secrets-sa@${PROJECT_ID}.iam.gserviceaccount.com \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[external-secrets/external-secrets]"

# 4. Annotate KSA inside the cluster
kubectl annotate serviceaccount external-secrets \
    --namespace external-secrets \
    iam.gke.io/gcp-service-account=external-secrets-sa@${PROJECT_ID}.iam.gserviceaccount.com \
    --overwrite
```

### Step 4: Deploy the ArgoCD Application (Bootstrap)
Render both the Application and the Repository Credentials Secret, passing your GitHub Personal Access Token (PAT) as a parameter:
```bash
helm template . -s templates/argocd-app.yaml -s templates/argocd-repo-secret.yaml --set gitops.password="<YOUR_GITHUB_PAT>" | kubectl apply -f -
```

---

## 🔍 Verification & Troubleshooting

### Check Pod Status
```bash
kubectl get pods -n todo-app
```

### Verify Secret Provisioning
Check if the External Secrets Operator successfully retrieved the secrets from Google Cloud and created the local `app-secrets` Secret:
```bash
kubectl get externalsecret app-secrets -n todo-app
kubectl get secret app-secrets -n todo-app -o yaml
```
