# How to Deploy Redis with Google Cloud Run

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Redis, Google Cloud, Cloud Run, Deployment, Container

Description: Deploy a self-managed Redis container on Google Cloud Run with a persistent Filestore volume, VPC connectivity, and Secret Manager for passwords.

---

Google Cloud Run is well-known for stateless containers, but with Cloud Run volumes backed by Cloud Filestore (NFS) or Cloud Storage, you can run stateful services like Redis for development and low-scale production workloads. This guide deploys Redis on Cloud Run with persistent storage and Secret Manager for secure password management.

## Prerequisites

```bash
# Enable required APIs
gcloud services enable \
  run.googleapis.com \
  secretmanager.googleapis.com \
  vpcaccess.googleapis.com \
  file.googleapis.com
```

## Store the Redis Password in Secret Manager

```bash
# Create the secret
echo -n "your-strong-password" | gcloud secrets create redis-password \
  --data-file=- \
  --replication-policy=automatic

# Grant Cloud Run access to the secret
gcloud secrets add-iam-policy-binding redis-password \
  --member="serviceAccount:$(gcloud projects describe $(gcloud config get-value project) \
    --format='value(projectNumber)')-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

## Create a Cloud Filestore Instance for Persistence

```bash
PROJECT=$(gcloud config get-value project)
REGION="us-central1"
ZONE="us-central1-a"

gcloud filestore instances create redis-storage \
  --project=$PROJECT \
  --location=$ZONE \
  --tier=BASIC_HDD \
  --file-share=name=redis_data,capacity=1TB \
  --network=name=default

# Get Filestore IP
FILESTORE_IP=$(gcloud filestore instances describe redis-storage \
  --location=$ZONE --format="value(networks[0].ipAddresses[0])")
```

## Deploy Redis to Cloud Run

```bash
gcloud run deploy redis \
  --image redis:7-alpine \
  --region $REGION \
  --platform managed \
  --no-allow-unauthenticated \
  --port 6379 \
  --memory 1Gi \
  --cpu 1 \
  --min-instances 1 \
  --max-instances 1 \
  --args="redis-server,--requirepass,$(gcloud secrets versions access latest --secret=redis-password),--maxmemory,512mb,--maxmemory-policy,allkeys-lru,--appendonly,yes,--dir,/data" \
  --vpc-connector projects/$PROJECT/locations/$REGION/connectors/my-connector \
  --vpc-egress all-traffic \
  --add-volume name=redis-vol,type=nfs,location=$FILESTORE_IP:/redis_data \
  --add-volume-mount volume=redis-vol,mount-path=/data
```

## Connecting from Other Cloud Run Services

Other Cloud Run services on the same VPC can connect by service URL:

```bash
# Get the internal URL
REDIS_URL=$(gcloud run services describe redis \
  --region $REGION --format="value(status.url)")

# Or connect via VPC internal address
# redis://:<password>@redis.internal:6379
```

Set the Redis URL as an environment variable or Secret Manager reference in dependent services:

```bash
gcloud run services update my-app \
  --region $REGION \
  --set-env-vars REDIS_HOST=10.x.x.x \
  --set-secrets REDIS_PASSWORD=redis-password:latest
```

## Verifying the Deployment

```bash
# Check service status
gcloud run services describe redis --region $REGION

# Stream logs
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=redis" \
  --limit 50 --format "value(textPayload)"
```

## Summary

Redis on Google Cloud Run uses a Cloud Filestore NFS volume for persistence and Secret Manager for password storage. VPC Connector ensures Redis is unreachable from the public internet while remaining accessible to other Cloud Run services. Setting min-instances to 1 prevents cold starts that would lose in-memory data between requests.
