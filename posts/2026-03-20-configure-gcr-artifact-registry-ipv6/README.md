# How to Configure GCR/Artifact Registry Access over IPv6

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: GCR, Google Artifact Registry, IPv6, GCP, Docker, Container Registry

Description: Configure Google Container Registry (GCR) and Artifact Registry to push and pull container images over IPv6 from GCP and external IPv6-capable infrastructure.

---

Google Artifact Registry (the successor to GCR) and GCR support IPv6 as Google's infrastructure is dual-stack. This guide covers configuring access from GCP VMs, external IPv6 hosts, and Kubernetes clusters.

## Checking IPv6 Support for Google Registries

```bash
# Check if Artifact Registry resolves to IPv6

dig AAAA us-docker.pkg.dev +short
dig AAAA gcr.io +short
dig AAAA us.gcr.io +short
dig AAAA eu.gcr.io +short

# Test IPv6 connectivity
curl -6 https://us-docker.pkg.dev/v2/ 2>&1
```

## Installing gcloud CLI and Configuring Auth

```bash
# Install gcloud CLI
curl https://sdk.cloud.google.com | bash
gcloud init

# Configure Docker authentication for GCR
gcloud auth configure-docker
# Or for Artifact Registry
gcloud auth configure-docker us-docker.pkg.dev

# Verify authentication
gcloud auth print-access-token | head -c 20
```

## Pulling Images from Artifact Registry over IPv6

```bash
# Authenticate
gcloud auth configure-docker us-docker.pkg.dev

# Pull from Artifact Registry
docker pull us-docker.pkg.dev/PROJECT_ID/REPO/IMAGE:TAG

# Verify IPv6 was used
curl -6 "https://us-docker.pkg.dev/v2/PROJECT_ID/REPO/IMAGE/tags/list" \
  -H "Authorization: Bearer $(gcloud auth print-access-token)"
```

## Pushing Images to Artifact Registry over IPv6

```bash
# Create an Artifact Registry repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="My Docker repository"

# Build and tag image for Artifact Registry
docker build -t myapp:latest .
docker tag myapp:latest \
  us-central1-docker.pkg.dev/PROJECT_ID/my-repo/myapp:latest

# Push the image
docker push us-central1-docker.pkg.dev/PROJECT_ID/my-repo/myapp:latest

# Monitor push progress
gcloud artifacts docker images list \
  us-central1-docker.pkg.dev/PROJECT_ID/my-repo
```

## Configuring GKE IPv6 Cluster for Artifact Registry

```bash
# Create GKE cluster with dual-stack support
gcloud container clusters create my-cluster \
  --enable-ip-alias \
  --cluster-ipv4-cidr="10.100.0.0/16" \
  --cluster-ipv6-cidr="2001:db8:1::/116" \
  --services-ipv6-cidr="2001:db8:2::/112" \
  --stack-type="IPV4_IPV6" \
  --zone=us-central1-a

# Configure kubectl
gcloud container clusters get-credentials my-cluster --zone=us-central1-a

# Deploy using Artifact Registry image
kubectl create deployment myapp \
  --image=us-central1-docker.pkg.dev/PROJECT_ID/my-repo/myapp:latest
```

## Using Workload Identity for Artifact Registry

```bash
# Create a Kubernetes service account
kubectl create serviceaccount artifact-sa

# Bind to Google Service Account
gcloud iam service-accounts create artifact-sa \
  --display-name="Artifact Registry Service Account"

gcloud projects add-iam-policy-binding PROJECT_ID \
  --member="serviceAccount:artifact-sa@PROJECT_ID.iam.gserviceaccount.com" \
  --role="roles/artifactregistry.reader"

# Annotate Kubernetes service account
kubectl annotate serviceaccount artifact-sa \
  iam.gke.io/gcp-service-account=artifact-sa@PROJECT_ID.iam.gserviceaccount.com
```

## Artifact Registry from External IPv6 Hosts

```bash
# From an external IPv6-capable host (non-GCP)
# Authenticate using service account key
gcloud auth activate-service-account \
  --key-file=/path/to/service-account-key.json

# Configure Docker authentication
gcloud auth configure-docker us-docker.pkg.dev

# Pull over IPv6 (if host has IPv6 and DNS returns AAAA)
docker pull us-docker.pkg.dev/PROJECT_ID/REPO/IMAGE:TAG

# Force IPv6 for connectivity testing
curl -6 -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  https://us-docker.pkg.dev/v2/PROJECT_ID/REPO/IMAGE/tags/list
```

## Troubleshooting Artifact Registry IPv6

```bash
# Verify DNS returns IPv6 addresses
dig AAAA us-docker.pkg.dev +short

# Test TCP connection over IPv6
nc -6 -w 5 us-docker.pkg.dev 443 && echo "Connected"

# Check authentication token
gcloud auth print-access-token 2>&1 | head -c 30

# Test with curl verbose
curl -6 -v \
  -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://us-docker.pkg.dev/v2/" 2>&1 | head -30
```

Google Artifact Registry's dual-stack infrastructure makes it naturally accessible over IPv6, requiring only proper authentication setup and IPv6-capable DNS to fully utilize IPv6 connectivity for container image management.
