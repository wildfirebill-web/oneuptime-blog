# How to Schedule Automatic Backups in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: portainer, backup, automation, business-edition, s3

Description: A guide to scheduling automatic backups in Portainer Business Edition, including S3 configuration, retention policies, and backup verification.

## Overview

Portainer Business Edition includes a built-in automatic backup scheduler that can export your Portainer configuration to S3-compatible storage on a schedule. This eliminates the need for manual backup scripts and ensures your configuration is always protected. This guide covers setting up scheduled backups, configuring S3 storage, and verifying backup integrity.

## Prerequisites

- Portainer Business Edition 2.x or newer
- Valid Portainer Business license
- S3-compatible storage (AWS S3, MinIO, Wasabi, etc.)
- Admin access to Portainer

## Automatic Backup Feature in Portainer BE

Portainer Business Edition provides:

- **Scheduled backups**: Cron-based scheduling
- **S3 upload**: Direct upload to S3-compatible storage
- **Password protection**: AES-encrypted backup archives
- **Retention management**: Automatic cleanup of old backups

## Step 1: Prepare S3 Storage

### AWS S3

```bash
# Create S3 bucket
aws s3 mb s3://portainer-backups --region us-east-1

# Create IAM policy
cat > portainer-backup-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::portainer-backups",
        "arn:aws:s3:::portainer-backups/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name PortainerBackupPolicy \
  --policy-document file://portainer-backup-policy.json
```

### MinIO (Self-Hosted S3)

```bash
# Deploy MinIO with Docker
docker run -d \
  -p 9000:9000 \
  -p 9001:9001 \
  --name minio \
  -v minio_data:/data \
  -e MINIO_ROOT_USER=minioadmin \
  -e MINIO_ROOT_PASSWORD=minioadmin \
  quay.io/minio/minio server /data --console-address ":9001"

# Create bucket using mc
docker run --rm \
  -e MC_HOST_local=http://minioadmin:minioadmin@minio:9000 \
  --link minio \
  minio/mc mb local/portainer-backups
```

## Step 2: Configure Automatic Backups in Portainer UI

1. Navigate to **Settings** → **Backup Portainer**
2. Toggle **Enable scheduled backups**
3. Configure the schedule:

| Field | Example | Description |
|---|---|---|
| Schedule | `0 2 * * *` | Daily at 2 AM UTC |
| Password | `SecureBackupPass123` | Encryption password |
| S3 compatible host | `s3.amazonaws.com` | S3 endpoint |
| Access key ID | `AKIAIOSFODNN7EXAMPLE` | AWS access key |
| Secret access key | `wJalrXUtnFEMI/K7MDENG` | AWS secret key |
| Region | `us-east-1` | S3 region |
| Bucket name | `portainer-backups` | Target bucket |

4. Click **Save backup schedule**

## Step 3: Common Cron Schedules

```bash
# Every 6 hours
0 */6 * * *

# Daily at 2 AM
0 2 * * *

# Weekly on Sunday at 3 AM
0 3 * * 0

# Every hour
0 * * * *
```

## Step 4: Manual Backup via API

```bash
PORTAINER_URL="https://portainer.example.com:9443"
TOKEN="your-api-token"

# Trigger immediate backup
curl -X POST \
  "${PORTAINER_URL}/api/backup" \
  -H "Authorization: Bearer ${TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"password": "BackupPassword123"}' \
  --output portainer-backup-$(date +%Y%m%d-%H%M%S).tar.gz
```

## Step 5: Verify Backups

```bash
# List backups in S3
aws s3 ls s3://portainer-backups/ --recursive

# Download and verify a backup
aws s3 cp s3://portainer-backups/portainer-backup-20260320.tar.gz ./

# Check backup integrity (Portainer uses AES encryption)
file portainer-backup-20260320.tar.gz
# Should show: gzip compressed data

# If password-protected, test restore in a staging environment
docker run --rm -v portainer_test:/data portainer/portainer-ee:latest \
  --restore-path /data/portainer-backup-20260320.tar.gz
```

## Step 6: Backup Retention with S3 Lifecycle Rules

```bash
# Set S3 lifecycle rule to expire old backups after 30 days
aws s3api put-bucket-lifecycle-configuration \
  --bucket portainer-backups \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "DeleteOldBackups",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Expiration": {"Days": 30}
    }]
  }'
```

## Monitoring Backup Status

```bash
# Check Portainer logs for backup activity
docker logs portainer 2>&1 | grep -i backup

# Set up CloudWatch alarm for missing backups (AWS)
aws cloudwatch put-metric-alarm \
  --alarm-name "PortainerBackupMissing" \
  --metric-name "NumberOfObjects" \
  --namespace "AWS/S3" \
  --statistic "Maximum" \
  --period 86400 \
  --threshold 1 \
  --comparison-operator "LessThanThreshold"
```

## Conclusion

Portainer Business Edition's built-in backup scheduler provides a reliable, low-maintenance solution for protecting your Portainer configuration. By combining scheduled S3 backups with lifecycle rules and monitoring, you can ensure your Portainer environment is always recoverable. Always test your backup restoration process periodically in a non-production environment to verify backup integrity.
