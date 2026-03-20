# How to Use Rancher with Google Cloud Marketplace

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, GCP, Google Cloud, Marketplace

Description: Deploy and manage Rancher through Google Cloud Marketplace on GKE, with integrated billing and GCP identity federation.

## Introduction

Google Cloud Marketplace offers SUSE Rancher as a click-to-deploy application on Google Kubernetes Engine (GKE). This enables quick deployment with GCP-consolidated billing and simplifies the procurement process for enterprises already invested in the Google Cloud ecosystem. This guide covers deploying Rancher on GKE through the Marketplace and configuring it for production use.

## Prerequisites

- A Google Cloud project with billing enabled
- `gcloud`, `kubectl`, and `helm` CLIs configured
- A domain and DNS zone managed in Google Cloud DNS

## Step 1: Subscribe via Google Cloud Marketplace

1. Navigate to [Google Cloud Marketplace → SUSE Rancher](https://console.cloud.google.com/marketplace).
2. Search for **"Rancher"**.
3. Click **Configure**.
4. Select your GCP project and deployment zone.
5. Click **Deploy** to proceed.

## Step 2: Create a GKE Cluster for Rancher

```bash
# Set project and zone

gcloud config set project my-project-id
gcloud config set compute/region us-central1

# Create a GKE cluster
gcloud container clusters create rancher-management \
  --zone us-central1-a \
  --num-nodes 3 \
  --machine-type n2-standard-4 \
  --release-channel stable \
  --enable-ip-alias \
  --enable-autoscaling \
  --min-nodes 3 \
  --max-nodes 6 \
  --workload-pool=my-project-id.svc.id.goog \
  --addons HorizontalPodAutoscaling,HttpLoadBalancing

# Get cluster credentials
gcloud container clusters get-credentials rancher-management \
  --zone us-central1-a

kubectl get nodes
```

## Step 3: Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true

kubectl rollout status deployment/cert-manager -n cert-manager --timeout=120s
```

## Step 4: Deploy Rancher

```bash
# Add Rancher Helm repo
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

# Create namespace
kubectl create namespace cattle-system

# Install Rancher with GKE-appropriate settings
helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=rancher.example.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=letsEncrypt \
  --set letsEncrypt.email=admin@example.com \
  --set replicas=3 \
  --set global.cattle.psp.enabled=false   # GKE 1.25+ removed PSPs

# Monitor rollout
kubectl rollout status deployment/rancher -n cattle-system --timeout=10m
```

## Step 5: Configure GCP Cloud DNS

```bash
# Get the Rancher ingress IP
RANCHER_IP=$(kubectl get ingress -n cattle-system rancher \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Rancher IP: ${RANCHER_IP}"

# Create a Cloud DNS record
gcloud dns record-sets create rancher.example.com. \
  --zone=example-zone \
  --type=A \
  --ttl=300 \
  --rrdatas="${RANCHER_IP}"
```

## Step 6: Enable Workload Identity for GCP Integration

Using Workload Identity avoids storing GCP service account keys:

```bash
# Create a GCP Service Account for Rancher
gcloud iam service-accounts create rancher-gke-sa \
  --display-name="Rancher GKE Service Account"

# Grant necessary GKE permissions
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:rancher-gke-sa@my-project-id.iam.gserviceaccount.com" \
  --role="roles/container.admin"

# Bind Kubernetes service account to GCP service account (Workload Identity)
gcloud iam service-accounts add-iam-policy-binding \
  rancher-gke-sa@my-project-id.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project-id.svc.id.goog[cattle-system/rancher]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount rancher \
  -n cattle-system \
  iam.gke.io/gcp-service-account=rancher-gke-sa@my-project-id.iam.gserviceaccount.com
```

## Step 7: Configure Google OIDC Authentication

```bash
# Create an OAuth 2.0 client ID in GCP Console
# APIs & Services → Credentials → Create Credentials → OAuth Client ID
# Application type: Web application
# Authorized redirect URIs: https://rancher.example.com/verify-auth

# In Rancher UI:
# Users & Authentication → Auth Provider → Google OAuth
# Client ID: <your-oauth-client-id>
# Client Secret: <your-oauth-client-secret>
# Click Enable
```

## Step 8: Monitor GCP Marketplace Billing

```bash
# View Rancher-related billing
gcloud billing accounts list

# Export billing data to BigQuery for analysis
bq query --use_legacy_sql=false '
SELECT
  service.description,
  SUM(cost) as total_cost,
  currency
FROM `my-project-id.billing_export.gcp_billing_export_v1_XXXXX`
WHERE service.description LIKE "%Rancher%"
  AND DATE(_PARTITIONTIME) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
GROUP BY service.description, currency
'
```

## Step 9: Enable Multi-Cluster Management

```bash
# Import additional GKE clusters into Rancher
# Create a second GKE cluster
gcloud container clusters create workload-cluster \
  --zone us-central1-b \
  --num-nodes 3 \
  --machine-type n2-standard-2

gcloud container clusters get-credentials workload-cluster \
  --zone us-central1-b

# In Rancher UI: Cluster Management → Import Existing → Generic
# Follow the kubectl command displayed to register the cluster
```

## Conclusion

Rancher on Google Cloud Marketplace provides a GCP-native deployment path with consolidated billing and easy integration with GCP identity services. Running Rancher on GKE with Workload Identity eliminates the need for service account key management, and GCP's managed control plane ensures the Rancher management cluster stays highly available. This setup is ideal for teams building a centralized multi-cluster management platform on Google Cloud.
