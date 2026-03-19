# How to Back Up Rancher to Google Cloud Storage

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Kubernetes, Backup, GCS

Description: Learn how to configure the Rancher Backup Operator to store backups in Google Cloud Storage using S3-compatible interoperability.

Google Cloud Storage (GCS) offers an S3-compatible interoperability mode that works seamlessly with the Rancher Backup Operator. This guide shows you how to set up GCS as a backup destination for your Rancher management server.

## Prerequisites

- Rancher v2.5 or later with the Backup Operator installed
- A Google Cloud account with a project
- gcloud CLI installed and configured
- kubectl access with cluster admin privileges

## Step 1: Create a GCS Bucket

Create a bucket for Rancher backups:

```bash
gsutil mb -p YOUR_PROJECT_ID -l us-central1 -b on gs://rancher-backups-production
```

Enable versioning for additional safety:

```bash
gsutil versioning set on gs://rancher-backups-production
```

## Step 2: Enable S3 Interoperability

GCS provides an S3-compatible API through its interoperability settings. Enable interop access in the Google Cloud Console:

1. Go to **Cloud Storage** > **Settings** > **Interoperability**.
2. Click **Create a key for a service account** or use the default project keys.
3. Note the **Access Key** and **Secret** that are generated.

Alternatively, create interoperability keys using gsutil:

```bash
gsutil hmac create YOUR_SERVICE_ACCOUNT@YOUR_PROJECT.iam.gserviceaccount.com
```

This outputs an access key and secret.

## Step 3: Create a Service Account with Proper Permissions

Create a dedicated service account for Rancher backups:

```bash
gcloud iam service-accounts create rancher-backup \
  --display-name="Rancher Backup Service Account" \
  --project=YOUR_PROJECT_ID
```

Grant the necessary permissions:

```bash
gsutil iam ch \
  serviceAccount:rancher-backup@YOUR_PROJECT_ID.iam.gserviceaccount.com:objectAdmin \
  gs://rancher-backups-production
```

Create HMAC keys for this service account:

```bash
gsutil hmac create rancher-backup@YOUR_PROJECT_ID.iam.gserviceaccount.com
```

Save the access key ID and secret from the output.

## Step 4: Create the Credentials Secret

Store the interoperability credentials as a Kubernetes secret:

```bash
kubectl create secret generic gcs-creds \
  -n cattle-resources-system \
  --from-literal=accessKey=GOOG1EEXAMPLEACCESSKEY \
  --from-literal=secretKey=YOUR_SECRET_KEY
```

## Step 5: Create a Backup Targeting GCS

Create the Backup resource using the GCS S3-compatible endpoint:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-backup-gcs
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 10
  storageLocation:
    s3:
      bucketName: rancher-backups-production
      folder: backups
      endpoint: storage.googleapis.com
      region: us-central1
      credentialSecretName: gcs-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply the backup:

```bash
kubectl apply -f backup-gcs.yaml
```

## Step 6: Create Scheduled Backups to GCS

For daily automated backups:

```yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-daily-gcs-backup
spec:
  resourceSetName: rancher-resource-set
  retentionCount: 30
  schedule: "0 2 * * *"
  storageLocation:
    s3:
      bucketName: rancher-backups-production
      folder: daily
      endpoint: storage.googleapis.com
      region: us-central1
      credentialSecretName: gcs-creds
      credentialSecretNamespace: cattle-resources-system
```

Apply the scheduled backup:

```bash
kubectl apply -f scheduled-backup-gcs.yaml
```

## Step 7: Verify Backups in GCS

Check the backup status:

```bash
kubectl get backups.resources.cattle.io rancher-backup-gcs -o yaml
```

List backups in the GCS bucket:

```bash
gsutil ls gs://rancher-backups-production/backups/
```

## Step 8: Configure Lifecycle Rules

Set up lifecycle rules to automatically manage old backups:

```bash
cat > lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {
          "type": "SetStorageClass",
          "storageClass": "NEARLINE"
        },
        "condition": {
          "age": 30
        }
      },
      {
        "action": {
          "type": "SetStorageClass",
          "storageClass": "COLDLINE"
        },
        "condition": {
          "age": 90
        }
      },
      {
        "action": {
          "type": "Delete"
        },
        "condition": {
          "age": 365
        }
      }
    ]
  }
}
EOF

gsutil lifecycle set lifecycle.json gs://rancher-backups-production
```

## Step 9: Enable Bucket Lock for Compliance

If compliance requires immutable backups, enable a retention policy:

```bash
gsutil retention set 30d gs://rancher-backups-production
```

This prevents any backup from being deleted for 30 days.

## Troubleshooting

### Authentication Errors

Verify the HMAC keys are active:

```bash
gsutil hmac list
```

If keys are inactive, create new ones and update the Kubernetes secret.

### Endpoint Issues

Make sure to use `storage.googleapis.com` as the endpoint. Do not include `https://` prefix.

### Permission Denied

Verify the service account has `objectAdmin` permissions on the bucket:

```bash
gsutil iam get gs://rancher-backups-production
```

### Region Configuration

GCS does not enforce region validation in the S3-compatible API the same way AWS does. You can use any valid GCS region string, but the bucket's actual location is determined when it was created.

## Conclusion

Using Google Cloud Storage with the Rancher Backup Operator through S3 interoperability provides a reliable backup solution within the Google Cloud ecosystem. With lifecycle management, versioning, and optional retention policies, you can build a comprehensive backup strategy that meets both operational and compliance requirements.
