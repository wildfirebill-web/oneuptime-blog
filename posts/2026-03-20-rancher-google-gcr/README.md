# How to Configure Google Container Registry with Rancher

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Google Cloud, GCR, Container Registry, Artifact Registry

Description: Set up Google Container Registry (GCR) or Artifact Registry with Rancher using service accounts and Workload Identity for secure container image management.

## Introduction

Google Container Registry (GCR) and its successor Artifact Registry are Google Cloud's managed container image storage services. Integrating them with Rancher requires proper IAM authentication. This guide covers using service account keys, Workload Identity for GKE, and configuring cluster-wide registry access.

## Prerequisites

- Google Cloud project with billing enabled
- GCR or Artifact Registry configured
- `gcloud` CLI installed and authenticated
- Rancher managing a GKE or other cluster
- kubectl access to your cluster

## Step 1: Create a Service Account for Registry Access

```bash
# Create a dedicated service account for registry pulls
gcloud iam service-accounts create rancher-registry-sa \
  --display-name="Rancher Registry Service Account" \
  --project=my-project-id

# Grant the registry reader role
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:rancher-registry-sa@my-project-id.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# For GCR (legacy), use storage.objectViewer
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:rancher-registry-sa@my-project-id.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

## Step 2: Create and Download a Service Account Key

```bash
# Create a JSON key for the service account
gcloud iam service-accounts keys create gcr-key.json \
  --iam-account=rancher-registry-sa@my-project-id.iam.gserviceaccount.com

# View the key file (contains sensitive credentials)
cat gcr-key.json
```

## Step 3: Create Kubernetes Secret for GCR

```bash
# Create the registry secret using the service account key
kubectl create secret docker-registry gcr-credentials \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat gcr-key.json)" \
  --docker-email=admin@example.com \
  --namespace=production

# For Artifact Registry in us-central1
kubectl create secret docker-registry ar-credentials \
  --docker-server=us-central1-docker.pkg.dev \
  --docker-username=_json_key \
  --docker-password="$(cat gcr-key.json)" \
  --docker-email=admin@example.com \
  --namespace=production
```

## Step 4: Configure Artifact Registry

```bash
# Create an Artifact Registry repository
gcloud artifacts repositories create my-containers \
  --repository-format=docker \
  --location=us-central1 \
  --description="Production container images"

# Authenticate local Docker to Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Build and push an image
docker build -t us-central1-docker.pkg.dev/my-project-id/my-containers/my-app:v1.0 .
docker push us-central1-docker.pkg.dev/my-project-id/my-containers/my-app:v1.0
```

## Step 5: Deploy Using GCR/Artifact Registry Images

```yaml
# deployment.yaml - Using Artifact Registry image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-gcp-app
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-gcp-app
  template:
    metadata:
      labels:
        app: my-gcp-app
    spec:
      # Reference the registry credential secret
      imagePullSecrets:
        - name: ar-credentials
      containers:
        - name: my-gcp-app
          # Artifact Registry image URI
          image: us-central1-docker.pkg.dev/my-project-id/my-containers/my-app:v1.0
          ports:
            - containerPort: 8080
```

## Step 6: Configure Workload Identity on GKE

For GKE clusters managed by Rancher, use Workload Identity to avoid key management:

```bash
# Enable Workload Identity on GKE cluster
gcloud container clusters update my-cluster \
  --workload-pool=my-project-id.svc.id.goog \
  --region=us-central1

# Create Kubernetes service account
kubectl create serviceaccount registry-puller \
  --namespace production

# Bind Kubernetes SA to GCP SA
gcloud iam service-accounts add-iam-policy-binding \
  rancher-registry-sa@my-project-id.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project-id.svc.id.goog[production/registry-puller]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount registry-puller \
  --namespace production \
  iam.gke.io/gcp-service-account=rancher-registry-sa@my-project-id.iam.gserviceaccount.com
```

```yaml
# workload-identity-pod.yaml - Pod using Workload Identity
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-wi
  namespace: production
spec:
  template:
    spec:
      # Use the annotated service account
      serviceAccountName: registry-puller
      containers:
        - name: my-app
          image: us-central1-docker.pkg.dev/my-project-id/my-containers/my-app:v1.0
```

## Step 7: Configure GCR with RKE2 Clusters

```yaml
# /etc/rancher/rke2/registries.yaml - RKE2 private registry config
mirrors:
  "gcr.io":
    endpoints:
      - "https://gcr.io"
  "us-central1-docker.pkg.dev":
    endpoints:
      - "https://us-central1-docker.pkg.dev"
configs:
  "gcr.io":
    auth:
      username: "_json_key"
      password: |
        {
          "type": "service_account",
          "project_id": "my-project-id",
          ...
        }
```

## Step 8: Set Up Automated Key Rotation

```bash
# Script to rotate service account key and update Kubernetes secret
#!/bin/bash
SA_EMAIL="rancher-registry-sa@my-project-id.iam.gserviceaccount.com"
NAMESPACE="production"
SECRET_NAME="gcr-credentials"

# Create new key
gcloud iam service-accounts keys create new-gcr-key.json \
  --iam-account=$SA_EMAIL

# Update Kubernetes secret
kubectl create secret docker-registry $SECRET_NAME \
  --docker-server=gcr.io \
  --docker-username=_json_key \
  --docker-password="$(cat new-gcr-key.json)" \
  --namespace=$NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

# Delete old keys (keep only 2 most recent)
OLD_KEYS=$(gcloud iam service-accounts keys list \
  --iam-account=$SA_EMAIL \
  --format="value(name)" | tail -n +3)

for KEY in $OLD_KEYS; do
  gcloud iam service-accounts keys delete $KEY \
    --iam-account=$SA_EMAIL --quiet
done
```

## Troubleshooting

```bash
# Verify service account has correct permissions
gcloud projects get-iam-policy my-project-id \
  --flatten="bindings[].members" \
  --filter="bindings.members:rancher-registry-sa"

# Test authentication
docker login -u _json_key -p "$(cat gcr-key.json)" gcr.io

# Check pod pull errors
kubectl describe pod <pod-name> -n production | grep -A 5 "Failed to pull"
```

## Conclusion

Google Container Registry and Artifact Registry integrate well with Rancher-managed clusters. For GKE clusters, Workload Identity provides the most secure approach by eliminating the need for long-lived service account keys. For non-GKE clusters, use service account keys with a rotation strategy. Artifact Registry is recommended over legacy GCR for new deployments as it offers more features and regional storage options.
