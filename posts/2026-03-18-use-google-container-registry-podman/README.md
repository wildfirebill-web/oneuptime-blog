# How to Use Google Container Registry with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Registry, Google Cloud, GCR, Artifact Registry

Description: Learn how to use Google Container Registry and Artifact Registry with Podman for managing container images on Google Cloud.

---

> Google Cloud offers both Container Registry and the newer Artifact Registry, and Podman works with both using service account authentication.

Google Container Registry (gcr.io) and its successor Google Artifact Registry provide managed container image hosting on Google Cloud Platform. Podman connects to these registries using GCP service account credentials or gcloud CLI authentication. This guide covers both registries and all authentication methods.

---

## Prerequisites

Install and configure the Google Cloud CLI.

```bash
# Install gcloud CLI (Linux)
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xf google-cloud-cli-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh

# Initialize gcloud
gcloud init

# Verify authentication
gcloud auth list

# Set the active project
gcloud config set project my-gcp-project
```

## Authenticating Podman to GCR

Use gcloud to generate an access token for Podman.

```bash
# Method 1: Use gcloud access token
gcloud auth print-access-token | podman login gcr.io \
  --username oauth2accesstoken \
  --password-stdin

# Method 2: Use gcloud credential helper output
gcloud auth configure-docker gcr.io 2>/dev/null
# This updates ~/.docker/config.json, which Podman can also read

# Method 3: Login to regional registries
gcloud auth print-access-token | podman login us.gcr.io \
  --username oauth2accesstoken \
  --password-stdin
```

## Using Service Account Keys

For CI/CD, use a service account JSON key file.

```bash
# Create a service account
gcloud iam service-accounts create podman-ci \
  --display-name "Podman CI Service Account"

# Grant the service account access to Container Registry
gcloud projects add-iam-policy-binding my-gcp-project \
  --member "serviceAccount:podman-ci@my-gcp-project.iam.gserviceaccount.com" \
  --role "roles/storage.admin"

# Create and download a key file
gcloud iam service-accounts keys create /tmp/sa-key.json \
  --iam-account podman-ci@my-gcp-project.iam.gserviceaccount.com

# Login to GCR using the service account key
cat /tmp/sa-key.json | podman login gcr.io \
  --username _json_key \
  --password-stdin

# For base64-encoded keys (common in CI/CD)
cat /tmp/sa-key.json | base64 | podman login gcr.io \
  --username _json_key_base64 \
  --password-stdin
```

## Pulling Images from GCR

Pull images from Google Container Registry.

```bash
# Pull from the global registry
podman pull gcr.io/my-gcp-project/myapp:latest

# Pull from regional registries (faster for specific regions)
podman pull us.gcr.io/my-gcp-project/myapp:latest
podman pull eu.gcr.io/my-gcp-project/myapp:latest
podman pull asia.gcr.io/my-gcp-project/myapp:latest

# Pull Google-provided images
podman pull gcr.io/google-containers/pause:3.9

# List pulled GCR images
podman images | grep gcr.io
```

## Pushing Images to GCR

Tag and push images to Google Container Registry.

```bash
PROJECT_ID="my-gcp-project"

# Build a local image
podman build -t myapp:latest .

# Tag for GCR
podman tag myapp:latest gcr.io/${PROJECT_ID}/myapp:latest
podman tag myapp:latest gcr.io/${PROJECT_ID}/myapp:v1.0

# Push the image
podman push gcr.io/${PROJECT_ID}/myapp:latest
podman push gcr.io/${PROJECT_ID}/myapp:v1.0

# List images in the repository
gcloud container images list --repository=gcr.io/${PROJECT_ID}
```

## Using Google Artifact Registry

Artifact Registry is the recommended successor to Container Registry.

```bash
# Create an Artifact Registry repository
gcloud artifacts repositories create my-docker-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="Docker images repository"

# Authenticate to Artifact Registry
gcloud auth print-access-token | podman login us-central1-docker.pkg.dev \
  --username oauth2accesstoken \
  --password-stdin

# Tag and push to Artifact Registry
PROJECT_ID="my-gcp-project"
AR_LOCATION="us-central1"
AR_REPO="my-docker-repo"

podman tag myapp:latest \
  ${AR_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/myapp:latest
podman push \
  ${AR_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/myapp:latest
```

## Managing GCR Images

List, inspect, and delete images using gcloud.

```bash
PROJECT_ID="my-gcp-project"

# List all images in the project
gcloud container images list --repository=gcr.io/${PROJECT_ID}

# List tags for a specific image
gcloud container images list-tags gcr.io/${PROJECT_ID}/myapp

# Describe an image
gcloud container images describe gcr.io/${PROJECT_ID}/myapp:latest

# Delete an image
gcloud container images delete gcr.io/${PROJECT_ID}/myapp:v1.0 --quiet
```

## Configuring GCR in registries.conf

Add GCR to your Podman configuration.

```toml
# /etc/containers/registries.conf

[[registry]]
prefix = "gcr.io"
location = "gcr.io"
insecure = false

[[registry]]
prefix = "us.gcr.io"
location = "us.gcr.io"
insecure = false
```

## CI/CD Pipeline with GCR

A complete CI/CD script for Google Cloud and Podman.

```bash
#!/bin/bash
# ci-gcr-deploy.sh - Build and push to Google Container Registry

set -euo pipefail

PROJECT_ID="${GCP_PROJECT_ID}"
IMAGE_NAME="myapp"
IMAGE_TAG="${CI_COMMIT_SHA:0:8}"
GCR_HOST="gcr.io"

# Authenticate using service account key
echo "${GCP_SA_KEY}" | podman login "${GCR_HOST}" \
  --username _json_key \
  --password-stdin

# Build and tag
podman build -t "${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}" .
podman tag "${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}" \
  "${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:latest"

# Push both tags
podman push "${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
podman push "${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:latest"

# Logout
podman logout "${GCR_HOST}"
echo "Deployed: ${GCR_HOST}/${PROJECT_ID}/${IMAGE_NAME}:${IMAGE_TAG}"
```

## Summary

Google Container Registry and Artifact Registry work with Podman through OAuth2 access tokens or service account JSON keys. Use `gcloud auth print-access-token` for interactive authentication and service account keys for CI/CD pipelines. GCR supports regional registries for optimized pull performance. Google recommends migrating to Artifact Registry for new projects, which uses a slightly different URL format but the same authentication mechanisms.
