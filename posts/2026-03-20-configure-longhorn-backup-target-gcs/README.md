# How to Configure Longhorn Backup Target to Google Cloud Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Backup, GCS, Google Cloud

Description: Configure Longhorn to use Google Cloud Storage (GCS) as a backup target for storing Kubernetes persistent volume backups in Google Cloud.

## Introduction

Google Cloud Storage (GCS) is Google's object storage service, providing high durability, scalability, and integration with the Google Cloud ecosystem. GCS is the natural choice for Longhorn backups on GKE clusters or in environments already using Google Cloud Platform. This guide covers the complete configuration process.

## Prerequisites

- Longhorn installed on your cluster
- Google Cloud project with billing enabled
- `gcloud` CLI installed and authenticated
- Network access from cluster nodes to `storage.googleapis.com`

## Step 1: Create a GCS Bucket

```bash
# Set your project and bucket configuration

PROJECT_ID="your-gcp-project-id"
BUCKET_NAME="longhorn-backups-$(echo $PROJECT_ID | tr -d -)"
REGION="us-central1"

# Create the GCS bucket
gsutil mb -p $PROJECT_ID \
  -c STANDARD \
  -l $REGION \
  gs://$BUCKET_NAME

# Enable uniform bucket-level access (recommended)
gsutil uniformbucketlevelaccess set on gs://$BUCKET_NAME

# Block public access
gsutil policyonly set gs://$BUCKET_NAME

echo "Bucket created: gs://$BUCKET_NAME"
```

## Step 2: Create a Service Account

```bash
# Create a dedicated service account for Longhorn backups
gcloud iam service-accounts create longhorn-backup-sa \
  --display-name="Longhorn Backup Service Account" \
  --project=$PROJECT_ID

# Get the service account email
SA_EMAIL="longhorn-backup-sa@${PROJECT_ID}.iam.gserviceaccount.com"
echo "Service account: $SA_EMAIL"

# Grant the service account access to the bucket
gsutil iam ch serviceAccount:${SA_EMAIL}:roles/storage.objectAdmin gs://$BUCKET_NAME
```

## Step 3: Create and Download Service Account Key

```bash
# Create a JSON key file for the service account
gcloud iam service-accounts keys create longhorn-backup-key.json \
  --iam-account=$SA_EMAIL \
  --project=$PROJECT_ID

echo "Key saved to longhorn-backup-key.json"
```

> **Security Note:** Keep the key file secure. Delete it after creating the Kubernetes secret.

## Step 4: Create Kubernetes Secret

Longhorn uses S3-compatible credentials for GCS access via the `GOOGLE_APPLICATION_CREDENTIALS` environment variable or by treating GCS as S3-compatible:

```bash
# Method 1: Using service account JSON key
kubectl create secret generic longhorn-backup-gcs \
  -n longhorn-system \
  --from-file=GOOGLE_APPLICATION_CREDENTIALS=longhorn-backup-key.json
```

```yaml
# longhorn-gcs-secret.yaml - GCS credentials for Longhorn
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-backup-gcs
  namespace: longhorn-system
type: Opaque
stringData:
  # The entire service account JSON key content
  GOOGLE_APPLICATION_CREDENTIALS: |
    {
      "type": "service_account",
      "project_id": "your-project-id",
      "private_key_id": "key-id",
      "private_key": "-----BEGIN RSA PRIVATE KEY-----\n...\n-----END RSA PRIVATE KEY-----\n",
      "client_email": "longhorn-backup-sa@your-project.iam.gserviceaccount.com",
      "client_id": "123456789",
      "auth_uri": "https://accounts.google.com/o/oauth2/auth",
      "token_uri": "https://oauth2.googleapis.com/token"
    }
```

```bash
kubectl apply -f longhorn-gcs-secret.yaml

# Delete the local key file after creating the secret
rm longhorn-backup-key.json
```

## Step 5: Configure Longhorn Backup Target

### Via kubectl

```bash
# Set GCS as the backup target
# Format: gs://bucket-name/optional-prefix
kubectl patch settings.longhorn.io backup-target \
  -n longhorn-system \
  --type merge \
  -p "{\"value\": \"gs://${BUCKET_NAME}/\"}"

# Set the credentials secret
kubectl patch settings.longhorn.io backup-target-credential-secret \
  -n longhorn-system \
  --type merge \
  -p '{"value": "longhorn-backup-gcs"}'
```

### Via Longhorn UI

1. Navigate to **Setting** → **General**
2. Find **Backup Target**
3. Enter: `gs://your-bucket-name/`
4. Find **Backup Target Credential Secret**
5. Enter: `longhorn-backup-gcs`
6. Click **Save**

## Using Workload Identity on GKE

For GKE clusters with Workload Identity enabled, you can avoid managing service account keys:

```bash
# Bind the Kubernetes service account to the GCP service account
gcloud iam service-accounts add-iam-policy-binding \
  "longhorn-backup-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[longhorn-system/longhorn-service-account]"

# Annotate the Kubernetes service account
kubectl annotate serviceaccount longhorn-service-account \
  -n longhorn-system \
  iam.gke.io/gcp-service-account="longhorn-backup-sa@${PROJECT_ID}.iam.gserviceaccount.com"
```

## Verify the Connection

```bash
# Check the backup target configuration
kubectl get settings.longhorn.io backup-target -n longhorn-system -o yaml

# Verify by checking backup volumes (should not show errors)
kubectl get backupvolumes.longhorn.io -n longhorn-system
```

## Create a Test Backup and Verify in GCS

```bash
# After triggering a backup from the Longhorn UI, verify it exists in GCS
gsutil ls gs://$BUCKET_NAME/backupstore/volumes/

# Check backup size
gsutil du -sh gs://$BUCKET_NAME/
```

## Set Up GCS Lifecycle Policies

Configure automatic object management for cost optimization:

```bash
# Create lifecycle configuration
cat > gcs-lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 365}
      }
    ]
  }
}
EOF

# Apply the lifecycle policy to the bucket
gsutil lifecycle set gcs-lifecycle.json gs://$BUCKET_NAME
```

## Configure Recurring Backups

```yaml
# recurring-gcs-backup.yaml - Daily backup to GCS
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: daily-gcs-backup
  namespace: longhorn-system
spec:
  cron: "0 3 * * *"
  task: "backup"
  retain: 30
  concurrency: 2
  labels:
    target: gcs
```

```bash
kubectl apply -f recurring-gcs-backup.yaml
```

## Conclusion

Google Cloud Storage provides an excellent backup target for Longhorn, especially in GCP-based Kubernetes environments. The combination of Workload Identity federation for authentication, GCS lifecycle policies for cost management, and Longhorn's recurring backup system creates a robust, automated backup solution. Always verify your backup restoration process periodically to ensure your backup data is valid and accessible.
