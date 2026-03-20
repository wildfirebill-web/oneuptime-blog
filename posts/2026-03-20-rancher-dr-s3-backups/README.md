# How to Configure Rancher DR with S3 Backups

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Rancher, Disaster-recovery, S3, Backup, Kubernetes, AWS

Description: Complete configuration guide for setting up Rancher disaster recovery using Amazon S3 or S3-compatible storage for reliable backup storage.

## Introduction

Amazon S3 and S3-compatible storage provide durable, highly-available storage for Rancher backups. With 99.999999999% durability and cross-region replication capabilities, S3 is an ideal backend for DR backups.

## Prerequisites

- Rancher v2.6+ with backup-restore-operator
- AWS account or S3-compatible storage (MinIO, Ceph, etc.)
- IAM permissions for S3 operations
- kubectl access to Rancher management cluster

## Step 1: Create S3 Bucket

```bash
# Create primary backup bucket

aws s3 mb s3://rancher-production-backups \
  --region us-east-1

# Enable versioning for additional protection
aws s3api put-bucket-versioning \
  --bucket rancher-production-backups \
  --versioning-configuration Status=Enabled

# Enable server-side encryption
aws s3api put-bucket-encryption \
  --bucket rancher-production-backups \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789:key/your-key-id"
      }
    }]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket rancher-production-backups \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

## Step 2: Configure Cross-Region Replication

```bash
# Create destination bucket in DR region
aws s3 mb s3://rancher-dr-backups-west \
  --region us-west-2

# Enable versioning on destination bucket (required for replication)
aws s3api put-bucket-versioning \
  --bucket rancher-dr-backups-west \
  --region us-west-2 \
  --versioning-configuration Status=Enabled

# Create IAM role for replication
cat > /tmp/replication-role.json << 'ROLEEOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
ROLEEOF

aws iam create-role \
  --role-name RancherS3ReplicationRole \
  --assume-role-policy-document file:///tmp/replication-role.json

# Configure replication rule
aws s3api put-bucket-replication \
  --bucket rancher-production-backups \
  --replication-configuration '{
    "Role": "arn:aws:iam::123456789:role/RancherS3ReplicationRole",
    "Rules": [{
      "ID": "rancher-dr-replication",
      "Status": "Enabled",
      "Destination": {
        "Bucket": "arn:aws:s3:::rancher-dr-backups-west",
        "StorageClass": "STANDARD_IA"
      }
    }]
  }'
```

## Step 3: Create IAM Policy for Backup Operator

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
        "s3:GetObjectVersion",
        "s3:ListBucketVersions"
      ],
      "Resource": "arn:aws:s3:::rancher-production-backups/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation"
      ],
      "Resource": "arn:aws:s3:::rancher-production-backups"
    }
  ]
}
```

## Step 4: Install Backup Operator

```bash
# Add Helm repo
helm repo add rancher-charts https://charts.rancher.io
helm repo update

# Install backup-restore-operator
helm install rancher-backup rancher-charts/rancher-backup \
  --namespace cattle-resources-system \
  --create-namespace \
  --set image.repository=rancher/backup-restore-operator \
  --set image.tag=v4.0.0 \
  --set persistence.enabled=false  # Use S3, not local storage
```

## Step 5: Create S3 Credentials Secret

```bash
# Create AWS credentials secret
kubectl create secret generic rancher-backup-s3-creds \
  --namespace cattle-resources-system \
  --from-literal=accessKey="YOUR_AWS_ACCESS_KEY_ID" \
  --from-literal=secretKey="YOUR_AWS_SECRET_ACCESS_KEY"
```

## Step 6: Configure Backup Resource

```yaml
# rancher-s3-backup.yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-s3-backup
  namespace: cattle-resources-system
spec:
  # Storage location configuration
  storageLocation:
    s3:
      bucketName: rancher-production-backups
      folder: rancher              # Subfolder within bucket
      region: us-east-1
      endpoint: s3.amazonaws.com   # Use custom endpoint for MinIO/Ceph
      endpointCA: ""               # CA cert for custom endpoints
      insecureTLSSkipVerify: false # Never skip in production
      credentialSecretName: rancher-backup-s3-creds
      credentialSecretNamespace: cattle-resources-system
  
  # Backup schedule (cron format)
  schedule: "0 */2 * * *"          # Every 2 hours
  
  # Retention
  retentionCount: 12               # Keep 12 backups (24 hours)
  
  # Encryption
  encryptionConfigSecretName: backup-encryption-key
```

## Step 7: Configure Encryption

```bash
# Generate encryption key
ENCRYPTION_KEY=$(openssl rand -base64 32)

# Create encryption config secret
kubectl create secret generic backup-encryption-key \
  --namespace cattle-resources-system \
  --from-literal=encryptionConfig="{\"encryptionKey\":\"$ENCRYPTION_KEY\"}"

# Save key to secure location (critical for restore!)
echo "ENCRYPTION KEY: $ENCRYPTION_KEY"
echo "Store this key in your password manager immediately!"
```

## Step 8: Configure S3 Lifecycle Policy

```bash
# Set lifecycle rules to manage old backups
aws s3api put-bucket-lifecycle-configuration \
  --bucket rancher-production-backups \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "cleanup-old-backups",
      "Status": "Enabled",
      "Filter": {"Prefix": "rancher/"},
      "Expiration": {"Days": 30},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 7},
      "Transitions": [{
        "Days": 7,
        "StorageClass": "STANDARD_IA"
      }]
    }]
  }'
```

## Using MinIO as S3-Compatible Backend

For on-premises deployments:

```yaml
# minio-backup.yaml
apiVersion: resources.cattle.io/v1
kind: Backup
metadata:
  name: rancher-minio-backup
spec:
  storageLocation:
    s3:
      bucketName: rancher-backups
      folder: prod
      region: us-east-1           # Required field even for MinIO
      endpoint: minio.internal:9000
      endpointCA: |               # MinIO's CA cert if using self-signed TLS
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      insecureTLSSkipVerify: false
      credentialSecretName: minio-credentials
  schedule: "0 * * * *"
  retentionCount: 24
```

## Monitoring Backup Health

```bash
# Check backup status
kubectl get backup -n cattle-resources-system

# View backup details
kubectl describe backup rancher-s3-backup -n cattle-resources-system

# Check backup operator logs
kubectl logs -n cattle-resources-system \
  -l app.kubernetes.io/name=rancher-backup \
  --tail=50
```

## Conclusion

S3-backed Rancher backups provide enterprise-grade durability and the foundation for reliable DR. With cross-region replication, encryption, and lifecycle policies, your backups are protected against data loss while remaining cost-effective. Combine these backup configurations with tested restore procedures for a complete DR solution.
