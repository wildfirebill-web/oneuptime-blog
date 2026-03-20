# How to Configure Longhorn Backup Target to S3

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Longhorn, Kubernetes, Storage, Backup, S3, AWS

Description: Step-by-step instructions for configuring Longhorn to use Amazon S3 or S3-compatible storage as a backup target for volume backups and disaster recovery.

## Introduction

Amazon S3 and S3-compatible object storage systems are the most popular backup target options for Longhorn. S3 provides scalable, durable object storage with lifecycle policies that are perfect for long-term backup retention. This guide covers configuring Longhorn with both AWS S3 and S3-compatible storage like MinIO.

## Prerequisites

- Longhorn installed and running
- An S3 bucket (AWS S3 or S3-compatible)
- IAM credentials with appropriate bucket permissions
- Network access from cluster nodes to the S3 endpoint

## Required S3 IAM Permissions

Create an IAM policy with the following permissions for your Longhorn backup user:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:AbortMultipartUpload",
        "s3:ListMultipartUploadParts"
      ],
      "Resource": [
        "arn:aws:s3:::your-longhorn-backup-bucket",
        "arn:aws:s3:::your-longhorn-backup-bucket/*"
      ]
    }
  ]
}
```

## Step 1: Create the S3 Bucket

```bash
# Using AWS CLI - create the S3 bucket

aws s3 mb s3://your-longhorn-backup-bucket --region us-east-1

# Enable versioning (optional but recommended)
aws s3api put-bucket-versioning \
  --bucket your-longhorn-backup-bucket \
  --versioning-configuration Status=Enabled

# Block public access
aws s3api put-public-access-block \
  --bucket your-longhorn-backup-bucket \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

## Step 2: Create Kubernetes Secret with S3 Credentials

```bash
# Create the backup credentials secret in the longhorn-system namespace
kubectl create secret generic longhorn-backup-s3 \
  -n longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID="AKIAIOSFODNN7EXAMPLE" \
  --from-literal=AWS_SECRET_ACCESS_KEY="wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY" \
  --from-literal=AWS_ENDPOINTS=""  # Empty for standard AWS S3
```

Or using a YAML file (store this file securely, never commit to version control):

```yaml
# longhorn-s3-secret.yaml - S3 credentials for Longhorn backups
apiVersion: v1
kind: Secret
metadata:
  name: longhorn-backup-s3
  namespace: longhorn-system
type: Opaque
stringData:
  # AWS Access Key ID
  AWS_ACCESS_KEY_ID: "AKIAIOSFODNN7EXAMPLE"
  # AWS Secret Access Key
  AWS_SECRET_ACCESS_KEY: "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
  # Leave empty for standard AWS S3
  AWS_ENDPOINTS: ""
```

```bash
kubectl apply -f longhorn-s3-secret.yaml
```

## Step 3: Configure Longhorn Backup Target

### Via kubectl

```bash
# Set the backup target URL (format: s3://bucket@region/optional-prefix/)
kubectl patch settings.longhorn.io backup-target \
  -n longhorn-system \
  --type merge \
  -p '{"value": "s3://your-longhorn-backup-bucket@us-east-1/"}'

# Set the credentials secret
kubectl patch settings.longhorn.io backup-target-credential-secret \
  -n longhorn-system \
  --type merge \
  -p '{"value": "longhorn-backup-s3"}'
```

### Via Longhorn UI

1. Navigate to **Setting** → **General**
2. Find **Backup Target**
3. Enter the S3 URL: `s3://your-longhorn-backup-bucket@us-east-1/`
4. Find **Backup Target Credential Secret**
5. Enter the secret name: `longhorn-backup-s3`
6. Click **Save**

## Configuring for S3-Compatible Storage (MinIO)

For MinIO or other S3-compatible endpoints:

```bash
# Create secret with MinIO credentials and custom endpoint
kubectl create secret generic longhorn-backup-minio \
  -n longhorn-system \
  --from-literal=AWS_ACCESS_KEY_ID="minioadmin" \
  --from-literal=AWS_SECRET_ACCESS_KEY="minioadmin" \
  --from-literal=AWS_ENDPOINTS="http://minio.minio-system.svc:9000" \
  --from-literal=AWS_CERT=""  # Custom CA cert for HTTPS MinIO
```

```bash
# Set MinIO as the backup target
kubectl patch settings.longhorn.io backup-target \
  -n longhorn-system \
  --type merge \
  -p '{"value": "s3://longhorn-backups@us-east-1/"}'

kubectl patch settings.longhorn.io backup-target-credential-secret \
  -n longhorn-system \
  --type merge \
  -p '{"value": "longhorn-backup-minio"}'
```

## Using IAM Roles for Service Accounts (IRSA) on EKS

For EKS clusters using IRSA, you can avoid storing credentials in secrets:

```bash
# Annotate the Longhorn service account with the IAM role ARN
kubectl annotate serviceaccount longhorn-service-account \
  -n longhorn-system \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/LonghornBackupRole

# Set backup target without credentials (IRSA handles auth)
kubectl patch settings.longhorn.io backup-target \
  -n longhorn-system \
  --type merge \
  -p '{"value": "s3://your-longhorn-backup-bucket@us-east-1/"}'
```

## Verify the Backup Target Connection

```bash
# Check backup target availability in Longhorn
kubectl get settings.longhorn.io backup-target -n longhorn-system -o yaml

# Look for the backup target being available
kubectl get backupvolumes.longhorn.io -n longhorn-system
```

In the Longhorn UI, navigate to **Backup** - if you see the backup list load without errors, the connection is working.

## Test the Backup

```bash
# Create a test backup to verify the S3 connection
# Via the UI: Volume → three-dot menu → Create Backup

# Or via kubectl, trigger a backup on a volume
kubectl label volumes.longhorn.io my-test-volume \
  -n longhorn-system \
  "recurring-job.longhorn.io/test-backup=enabled"
```

## Configure S3 Lifecycle Policies

Set up S3 lifecycle rules to automatically expire old backups:

```bash
# Create a lifecycle configuration file
cat > lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "expire-old-backups",
      "Status": "Enabled",
      "Filter": {
        "Prefix": ""
      },
      "Expiration": {
        "Days": 90
      }
    }
  ]
}
EOF

# Apply the lifecycle policy to the S3 bucket
aws s3api put-bucket-lifecycle-configuration \
  --bucket your-longhorn-backup-bucket \
  --lifecycle-configuration file://lifecycle.json
```

## Conclusion

Configuring Longhorn with S3 backup targets provides a scalable, durable, and cost-effective solution for off-cluster backup storage. Whether using AWS S3 or an S3-compatible service like MinIO, the setup process is straightforward. Combined with recurring backup jobs, S3-backed Longhorn backups give you reliable disaster recovery capabilities for your Kubernetes persistent volumes.
