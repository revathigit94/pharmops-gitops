# PharmOps — GKE Deployment Guide

> **Note:** This is a forked version of [ravdy/pharmops](https://github.com/ravdy/pharmops) with modifications to deploy on **Google Kubernetes Engine (GKE)**. The original repo contains Terraform-based AWS EKS deployment steps. This guide covers GCP deployment only.

---

## Overview

PharmOps is a pharmaceutical microservices platform consisting of 5 services:

| Service | Tech | Port |
|---|---|---|
| pharma-ui | React + Nginx | 80 |
| api-gateway | Spring Boot | 8080 |
| auth-service | Spring Boot | 8081 |
| catalog-service | Spring Boot | 8082 |
| notification-service | Node.js | 3000 |

---

## Prerequisites

- GCP project with billing enabled
- GCP Cloud Shell or `gcloud` CLI installed locally
- `kubectl`, `helm`, `docker` available
- Basic Kubernetes knowledge

---

## Architecture on GKE

```
Browser
  └── NodePort 30080
        └── Nginx Ingress Controller
              ├── pharma-ui (port 80)
              └── api-gateway (port 8080)
                    ├── auth-service (port 8081)
                    │     └── Cloud SQL Auth Proxy sidecar → Cloud SQL
                    ├── catalog-service (port 8082)
                    │     └── Cloud SQL Auth Proxy sidecar → Cloud SQL
                    └── notification-service (port 3000)
                          └── Cloud SQL Auth Proxy sidecar → Cloud SQL
```

---

## Step 1: Setup GCP Project & Clone Repos

Open **GCP Cloud Shell** and run:

```bash
# Set your GCP project
gcloud config set project YOUR_PROJECT_ID

# Clone both repos
git clone https://github.com/revathigit94/pharmops.git
git clone https://github.com/revathigit94/pharmops-gitops.git
```

---

## Step 2: Create GKE Cluster

```bash
# Enable required APIs
gcloud services enable container.googleapis.com \
  artifactregistry.googleapis.com \
  sqladmin.googleapis.com \
  secretmanager.googleapis.com \
  compute.googleapis.com

# Create cluster
gcloud container clusters create pharmops-cluster \
  --zone asia-south1-a \
  --num-nodes 2 \
  --machine-type e2-standard-2 \
  --enable-autoscaling \
  --min-nodes 2 \
  --max-nodes 4

# Connect kubectl
gcloud container clusters get-credentials pharmops-cluster \
  --zone asia-south1-a

# Verify
kubectl get nodes
```

> **Note:** If you run into resource issues later, add a larger node pool:
> ```bash
> gcloud container node-pools create pharmops-pool-2 \
>   --cluster=pharmops-cluster \
>   --zone=asia-south1-a \
>   --num-nodes=2 \
>   --machine-type=e2-standard-4 \
>   --scopes="https://www.googleapis.com/auth/cloud-platform"
> ```

---

## Step 3: Create Artifact Registry & Push Images

```bash
export PROJECT_ID=$(gcloud config get-value project)
export REGION=asia-south1
export REGISTRY=$REGION-docker.pkg.dev/$PROJECT_ID/pharma-repo

# Create registry
gcloud artifacts repositories create pharma-repo \
  --repository-format=docker \
  --location=$REGION

# Authenticate Docker
gcloud auth configure-docker $REGION-docker.pkg.dev

# Build and push all 5 images
cd ~/pharmops

docker build -t $REGISTRY/pharma-ui:v1.0.0 ./services/pharma-ui
docker push $REGISTRY/pharma-ui:v1.0.0

docker build -t $REGISTRY/api-gateway:v1.0.0 ./services/api-gateway
docker push $REGISTRY/api-gateway:v1.0.0

docker build -t $REGISTRY/auth-service:v1.0.0 ./services/auth-service
docker push $REGISTRY/auth-service:v1.0.0

docker build -t $REGISTRY/catalog-service:v1.0.0 ./services/catalog-service
docker push $REGISTRY/catalog-service:v1.0.0

docker build -t $REGISTRY/notification-service:v1.0.0 ./services/notification-service
docker push $REGISTRY/notification-service:v1.0.0
```

---

## Step 4: Create Cloud SQL (PostgreSQL)

```bash
# Create instance
gcloud sql instances create pharmops-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=asia-south1 \
  --root-password=Admin1234!

# Create database and user
gcloud sql databases create pharma --instance=pharmops-db
gcloud sql users create pharmauser \
  --instance=pharmops-db \
  --password=Pharma1234!

# Save connection name
export DB_INSTANCE_CONNECTION=$(gcloud sql instances describe pharmops-db \
  --format="value(connectionName)")
echo $DB_INSTANCE_CONNECTION
```

### Initialize DB Schema

```bash
gcloud sql connect pharmops-db --user=pharmauser --database=pharma
```

Once connected, run these SQL commands first to fix role dependencies:

```sql
CREATE ROLE pharmaadmin;
GRANT pharmaadmin TO pharmauser;
\q
```

Then run the schema script:

```bash
gcloud sql connect pharmops-db --user=pharmauser --database=pharma \
  < ~/pharmops-gitops/db-init/01-schemas.sql
```

---

## Step 5: Configure Workload Identity & Service Accounts

```bash
# Create GCP service account
gcloud iam service-accounts create pharma-sa \
  --display-name="PharmOps Service Account"

# Grant required roles
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:pharma-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/cloudsql.client"

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:pharma-sa@$PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

gcloud iam service-accounts add-iam-policy-binding \
  pharma-sa@$PROJECT_ID.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:$PROJECT_ID.svc.id.goog[dev/pharma-sa]"

# Create Kubernetes namespace and service account
kubectl create namespace dev

kubectl create serviceaccount pharma-sa -n dev

kubectl annotate serviceaccount pharma-sa -n dev \
  iam.gke.io/gcp-service-account=pharma-sa@$PROJECT_ID.iam.gserviceaccount.com
```

---

## Step 6: Create Kubernetes Secrets

```bash
kubectl create secret generic db-credentials \
  --namespace=dev \
  --from-literal=DB_USERNAME=pharmauser \
  --from-literal=DB_PASSWORD=Pharma1234! \
  --from-literal=DB_NAME=pharma \
  --from-literal=DB_PORT=5432

kubectl create secret generic jwt-secret \
  --namespace=dev \
  --from-literal=JWT_SECRET=pharmops-super-secret-jwt-key-2024
```

---

## Step 7: Update Kubernetes Manifests

Update the following files in `pharmops-gitops/`:

### 7.1 All Helm values files (`envs/dev/values-*.yaml`)

| Field | Change |
|---|---|
| `image.repository` | Replace ECR URL with `$REGISTRY/<service-name>` |
| `configmap.DB_HOST` | Replace RDS endpoint with `"127.0.0.1"` |
| `serviceAccount.create` | Set to `false` |
| `serviceAccount.name` | Set to `pharma-sa` |

Also add to each values file that needs DB access (auth, catalog, notification):

```yaml
cloudsql:
  enabled: true
  instanceConnectionName: YOUR_PROJECT:REGION:pharmops-db
```

For api-gateway (no DB needed):

```yaml
cloudsql:
  enabled: false
  instanceConnectionName: ''
```

### 7.2 pharma-ui manifests (`k8s-manifests/pharma-ui/`)

In `deployment.yaml`:
- Replace ECR image URL with `$REGISTRY/pharma-ui:v1.0.0`
- Change `serviceAccountName` to `pharma-sa`

In `ingress.yaml`:
- Replace `<YOUR_ALB_HOSTNAME>` with `NODE_IP.nip.io`
- Set `nginx.ingress.kubernetes.io/ssl-redirect: "false"`

### 7.3 Update Helm deployment template

Add Cloud SQL Auth Proxy sidecar to `helm-charts/templates/deployment.yaml` inside the `containers:` block:

```yaml
{{- if .Values.cloudsql.enabled }}
- name: cloud-sql-proxy
  image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.8.0
  args:
    - "--structured-logs"
    - "--port=5432"
    - {{ .Values.cloudsql.instanceConnectionName | quote }}
  securityContext:
    runAsNonRoot: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "50m"
    limits:
      memory: "256Mi"
      cpu: "100m"
{{- end }}
```

---

## Step 8: Install Nginx Ingress Controller

> **Note:** If your GCP project has org policy restrictions on Load Balancer creation, use NodePort mode.

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443

# Get Node IP
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
echo $NODE_IP
```

### Create GCP Firewall Rule

```bash
gcloud compute firewall-rules create allow-nodeport \
  --allow=tcp:30080,tcp:30443 \
  --source-ranges=0.0.0.0/0 \
  --description="Allow NodePort access"
```

---

## Step 9: Deploy Services with Helm

```bash
cd ~/pharmops-gitops

# Deploy all services
helm install api-gateway helm-charts \
  -f envs/dev/values-api-gateway.yaml -n dev

helm install auth-service helm-charts \
  -f envs/dev/values-auth-service.yaml -n dev

helm install catalog-service helm-charts \
  -f envs/dev/values-catalog-service.yaml -n dev

helm install notification-service helm-charts \
  -f envs/dev/values-notification-service.yaml -n dev

# Deploy pharma-ui with raw manifests
kubectl apply -f k8s-manifests/pharma-ui/ -n dev

# Verify all pods are running
kubectl get pods -n dev
```

---

## Step 10: Access the Application

```bash
echo "App URL: http://$NODE_IP:30080"
```

Open `http://NODE_IP:30080` in your browser.

Default credentials:
- **Username:** `admin`
- **Password:** `admin123`

---

## Common Issues & Fixes

| Issue | Fix |
|---|---|
| LB stuck in Pending (org policy) | Use NodePort mode for Nginx Ingress |
| `Connection refused 127.0.0.1:5432` | Cloud SQL Auth Proxy sidecar not injected — check cloudsql values |
| `ACCESS_TOKEN_SCOPE_INSUFFICIENT` | Create node pool with `--scopes=https://www.googleapis.com/auth/cloud-platform` |
| Ingress rejects IP as hostname | Use `NODE_IP.nip.io` instead of raw IP |
| `nil pointer evaluating cloudsql.enabled` | Add `cloudsql: enabled: false` to values files missing it |
| `pharmaadmin role does not exist` | Run `CREATE ROLE pharmaadmin; GRANT pharmaadmin TO pharmauser;` in DB |
| Pods in Pending (insufficient resources) | Add a larger node pool (e2-standard-4) |
| Firewall rule missing | Recreate with `gcloud compute firewall-rules create allow-nodeport` |

---

## Tech Stack

`GKE` `Helm` `ArgoCD` `Docker` `Cloud SQL` `Artifact Registry` `Workload Identity` `Cloud SQL Auth Proxy` `Nginx Ingress` `Spring Boot` `React` `Node.js` `PostgreSQL`

---

## Credits

Original project: [ravdy/pharmops](https://github.com/ravdy/pharmops) — AWS EKS deployment  
This fork: GKE deployment adaptation
