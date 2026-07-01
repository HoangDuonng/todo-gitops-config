# Todo App GKE GitOps & Helm Chart

This repository is structured as a Helm Chart and ArgoCD configuration optimized for deploying the Todo application on a GKE cluster.

Secrets (passwords, tokens) are managed using the **External Secrets Operator (ESO)** and **Google Cloud Secret Manager (GCP SM)** to ensure they are never committed to Git in plain text.

---

## Repository Structure

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

## Bootstrap & Installation Guide

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

## Verification & Troubleshooting

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

---

## Cost Optimization (Cluster Hibernation)

To save on GCP VM costs when you are not using the cluster (saving up to 70% of your bill), you can temporarily hibernate the GKE cluster by scaling it down to 0 VMs (this will NOT delete your database persistent storage) and scaling it back up when you need it:

### Scale Down Command (Hibernate - 0 Nodes):

```bash
gcloud container clusters resize todo-cluster --num-nodes=0 --zone=asia-southeast1-a
```

### Scale Up Command (Resume - 3 Nodes):

```bash
gcloud container clusters resize todo-cluster --num-nodes=3 --zone=asia-southeast1-a
```

---

## Monitoring & Logging (Observability)

We deploy **kube-prometheus-stack** (Prometheus & Grafana) and **loki-stack** (Loki & Promtail) to provide centralized logging, dashboards, and metrics.

### Step 1: Deploy Prometheus & Grafana (kube-prometheus-stack)

Apply the customized lightweight configuration values:

```bash
# 1. Create monitoring namespace
kubectl create namespace monitoring || true

# 2. Add Helm charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 3. Deploy Prometheus & Grafana
helm upgrade --install prometheus prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f monitoring/prometheus-values.yaml
```

### Step 2: Deploy Loki & Promtail (loki-stack)

```bash
# Deploy Loki & Promtail
helm upgrade --install loki grafana/loki-stack \
  -n monitoring \
  -f monitoring/loki-values.yaml
```

### Step 3: Access Grafana UI & Query Logs

1. **Get Grafana Public IP:**

   ```bash
   kubectl get svc prometheus-grafana -n monitoring -w
   ```

   Once the `EXTERNAL-IP` changes from `<pending>` to a public IP, open it in your browser:
   **`http://<EXTERNAL_IP>`** (by default exposed on port 80).

2. **Retrieve Grafana Credentials:**
   - **Username**: `admin`
   - **Password**: Get the decoded password using the command:
     - _On PowerShell:_
       ```powershell
       [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}")))
       ```
     - _On Bash:_
       ```bash
       kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
       ```

3. **Query Microservices Logs (Loki):**
   - In Grafana, click the menu and navigate to **Explore**.
   - Choose the **Loki** data source from the dropdown.
   - Run LogQL queries to view your app logs, e.g., to query all logs in the `todo-app` namespace:
     ```logql
     {namespace="todo-app"}
     ```
     Or query a specific microservice:
     ```logql
     {namespace="todo-app", pod=~"auth-service-.*"}
     ```

---

## 🧹 Resource & History Retention (Best Practices)

To prevent resource and storage bloat on both GCP and GKE, we apply strict history limits and log retention:

### 1. K8s & ArgoCD Revision History Limits
All Deployments, StatefulSets, and the ArgoCD Application are configured with `revisionHistoryLimit: 5` to keep only the 5 most recent versions for rollbacks.

### 2. Loki Log Retention (2 Days)
Loki is configured to delete logs older than **48 hours (2 days)**. To apply this, re-run:
```bash
helm upgrade --install loki grafana/loki-stack \
  -n monitoring \
  -f monitoring/loki-values.yaml
```

### 3. Google Artifact Registry Cleanup Policy
We keep only the **5 most recent image tags** in the Artifact Registry. Apply this policy using the command:
```bash
gcloud artifacts repositories set-cleanup-policies todo-repo \
    --project=todo-devops-500719 \
    --location=asia-southeast1 \
    --policy=gcp/gar-cleanup-policy.json
```

