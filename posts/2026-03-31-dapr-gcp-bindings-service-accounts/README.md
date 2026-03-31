# How to Use Dapr GCP Bindings with Service Accounts

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Dapr, GCP, Service Account, Binding, IAM

Description: Learn how to create and configure GCP service accounts with minimal permissions for Dapr bindings, including key management, role assignments, and rotation strategies.

---

## GCP Service Accounts for Dapr Bindings

A GCP Service Account is an identity for non-human principals - like Dapr sidecars - to authenticate to GCP APIs. Properly scoped service accounts give Dapr bindings only the permissions they need, applying the principle of least privilege.

## Creating a Dedicated Service Account per Service

Avoid sharing service accounts between multiple Dapr applications. Create one per service or binding type:

```bash
# Create service accounts for different services
gcloud iam service-accounts create dapr-order-service \
  --display-name "Dapr Order Service" \
  --project my-gcp-project

gcloud iam service-accounts create dapr-analytics-service \
  --display-name "Dapr Analytics Service" \
  --project my-gcp-project

gcloud iam service-accounts create dapr-notification-service \
  --display-name "Dapr Notification Service" \
  --project my-gcp-project
```

## Assigning Minimal IAM Roles

### Storage Bucket Permissions

```bash
# Read-only access for analytics service
gcloud storage buckets add-iam-policy-binding gs://data-lake \
  --member "serviceAccount:dapr-analytics-service@my-gcp-project.iam.gserviceaccount.com" \
  --role "roles/storage.objectViewer"

# Write access for order service
gcloud storage buckets add-iam-policy-binding gs://order-attachments \
  --member "serviceAccount:dapr-order-service@my-gcp-project.iam.gserviceaccount.com" \
  --role "roles/storage.objectCreator"
```

### Pub/Sub Permissions

```bash
# Publisher only (cannot consume)
gcloud pubsub topics add-iam-policy-binding order-events \
  --member "serviceAccount:dapr-order-service@my-gcp-project.iam.gserviceaccount.com" \
  --role "roles/pubsub.publisher"

# Subscriber only for analytics (cannot publish)
gcloud pubsub subscriptions add-iam-policy-binding analytics-sub \
  --member "serviceAccount:dapr-analytics-service@my-gcp-project.iam.gserviceaccount.com" \
  --role "roles/pubsub.subscriber"
```

## Creating and Rotating Service Account Keys

### Creating the Key

```bash
gcloud iam service-accounts keys create \
  ./order-service-key.json \
  --iam-account dapr-order-service@my-gcp-project.iam.gserviceaccount.com

# List existing keys
gcloud iam service-accounts keys list \
  --iam-account dapr-order-service@my-gcp-project.iam.gserviceaccount.com
```

### Storing the Key in Kubernetes

Extract individual fields for use as separate secret values:

```bash
export KEY_FILE=./order-service-key.json
kubectl create secret generic gcp-order-service-key \
  --from-literal=privateKeyId=$(jq -r '.private_key_id' $KEY_FILE) \
  --from-literal=privateKey="$(jq -r '.private_key' $KEY_FILE)" \
  --from-literal=clientId=$(jq -r '.client_id' $KEY_FILE)
```

### Configuring the Binding Component

```yaml
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: order-storage
  namespace: default
spec:
  type: bindings.gcp.bucket
  version: v1
  metadata:
    - name: bucket
      value: "order-attachments"
    - name: type
      value: "service_account"
    - name: project_id
      value: "my-gcp-project"
    - name: private_key_id
      secretKeyRef:
        name: gcp-order-service-key
        key: privateKeyId
    - name: private_key
      secretKeyRef:
        name: gcp-order-service-key
        key: privateKey
    - name: client_email
      value: "dapr-order-service@my-gcp-project.iam.gserviceaccount.com"
    - name: client_id
      secretKeyRef:
        name: gcp-order-service-key
        key: clientId
    - name: auth_uri
      value: "https://accounts.google.com/o/oauth2/auth"
    - name: token_uri
      value: "https://oauth2.googleapis.com/token"
```

## Key Rotation Procedure

GCP recommends rotating keys every 90 days:

```bash
# Step 1: Create a new key
gcloud iam service-accounts keys create \
  ./order-service-key-new.json \
  --iam-account dapr-order-service@my-gcp-project.iam.gserviceaccount.com

# Step 2: Update the Kubernetes secret with the new key
kubectl create secret generic gcp-order-service-key \
  --from-literal=privateKeyId=$(jq -r '.private_key_id' ./order-service-key-new.json) \
  --from-literal=privateKey="$(jq -r '.private_key' ./order-service-key-new.json)" \
  --from-literal=clientId=$(jq -r '.client_id' ./order-service-key-new.json) \
  --dry-run=client -o yaml | kubectl apply -f -

# Step 3: Restart pods to pick up the new secret
kubectl rollout restart deployment/order-service

# Step 4: Delete the old key
OLD_KEY_ID="<old-key-id>"
gcloud iam service-accounts keys delete $OLD_KEY_ID \
  --iam-account dapr-order-service@my-gcp-project.iam.gserviceaccount.com
```

## Auditing Service Account Usage

```bash
# List all keys and their creation dates
gcloud iam service-accounts keys list \
  --iam-account dapr-order-service@my-gcp-project.iam.gserviceaccount.com \
  --format="table(name, validAfterTime, validBeforeTime)"

# Check recent API activity via Cloud Audit Logs
gcloud logging read \
  'protoPayload.authenticationInfo.principalEmail="dapr-order-service@my-gcp-project.iam.gserviceaccount.com"' \
  --limit=20 \
  --format="table(timestamp, protoPayload.methodName, protoPayload.resourceName)"
```

## Summary

GCP service accounts for Dapr bindings should follow least privilege principles - one account per service, with access only to the specific resources and operations needed. Store key fields in Kubernetes secrets using Dapr secret references, rotate keys every 90 days, and monitor usage via Cloud Audit Logs. For GKE deployments, replace key files entirely with Workload Identity to eliminate the rotation burden.
