# How to Schedule Automatic Backups to S3 in Portainer Business Edition

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Backup, S3, AWS, Business Edition, Automation

Description: Configure automated scheduled backups of Portainer Business Edition to Amazon S3 or S3-compatible storage for disaster recovery and compliance.

## Introduction

Portainer Business Edition includes built-in scheduled backup functionality that can automatically push encrypted backup archives to Amazon S3 or any S3-compatible storage (MinIO, Backblaze B2, Cloudflare R2). This ensures your Portainer configuration is automatically protected without manual intervention.

## Prerequisites

- Portainer Business Edition (BE) 2.17+
- An S3 bucket (AWS S3, MinIO, Backblaze B2, or Cloudflare R2)
- AWS IAM credentials with S3 write permissions
- Portainer admin access

## Step 1: Create an S3 Bucket

```bash
# Using AWS CLI
aws s3 mb s3://my-portainer-backups --region us-east-1

# Enable versioning (recommended for backups)
aws s3api put-bucket-versioning \
  --bucket my-portainer-backups \
  --versioning-configuration Status=Enabled

# Set lifecycle policy to delete old backups
cat > /tmp/lifecycle.json << 'EOF'
{
  "Rules": [
    {
      "ID": "delete-old-backups",
      "Status": "Enabled",
      "Filter": {"Prefix": "portainer/"},
      "Expiration": {"Days": 30},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 7}
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-portainer-backups \
  --lifecycle-configuration file:///tmp/lifecycle.json
```

## Step 2: Create an IAM User for Portainer Backups

```bash
# Create a dedicated IAM user
aws iam create-user --user-name portainer-backup

# Create a policy allowing only the necessary S3 actions
cat > /tmp/portainer-backup-policy.json << 'EOF'
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
        "arn:aws:s3:::my-portainer-backups",
        "arn:aws:s3:::my-portainer-backups/*"
      ]
    }
  ]
}
EOF

aws iam put-user-policy \
  --user-name portainer-backup \
  --policy-name portainer-backup-policy \
  --policy-document file:///tmp/portainer-backup-policy.json

# Create access keys
aws iam create-access-key --user-name portainer-backup
# Save the AccessKeyId and SecretAccessKey
```

## Step 3: Configure S3 Backup in Portainer UI

1. Log in to Portainer Business Edition
2. Go to **Settings** → **Backup**
3. Select **Automated backups**
4. Configure:
   - **S3 Compatible Host**: (leave empty for AWS, or enter endpoint for MinIO)
   - **Region**: your S3 region (e.g., `us-east-1`)
   - **Bucket**: `my-portainer-backups`
   - **Credentials**: Your AWS Access Key ID and Secret Access Key
   - **Schedule**: cron expression for when to run backups
   - **Backup Prefix**: `portainer/` (optional, for organization)
   - **Password**: encryption password for backup archive
5. Click **Save settings**
6. Click **Backup now** to test

## Step 4: Configure via API

```bash
TOKEN=$(curl -s -X POST http://localhost:9000/api/auth \
  -H "Content-Type: application/json" \
  -d '{"Username":"admin","Password":"yourpassword"}' | jq -r .jwt)

# Configure S3 backup settings
curl -X PUT \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  http://localhost:9000/api/backup/s3/settings \
  -d '{
    "accessKeyID": "AKIAIOSFODNN7EXAMPLE",
    "secretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "region": "us-east-1",
    "bucketName": "my-portainer-backups",
    "s3CompatibleHost": "",
    "scheduleEnabled": true,
    "cronExpression": "0 2 * * *",
    "password": "your-encryption-password"
  }'
```

## Step 5: Configure for S3-Compatible Storage

### MinIO

```bash
# In Portainer UI:
# S3 Compatible Host: http://minio.yourdomain.com:9000
# Region: us-east-1 (or any value for MinIO)
# Bucket: portainer-backups
# Access Key: your-minio-access-key
# Secret Key: your-minio-secret-key
```

### Backblaze B2

```bash
# In Portainer UI:
# S3 Compatible Host: https://s3.us-west-004.backblazeb2.com
# Region: us-west-004 (your B2 region)
# Bucket: your-b2-bucket-name
# Access Key: your-b2-key-id
# Secret Key: your-b2-application-key
```

### Cloudflare R2

```bash
# In Portainer UI:
# S3 Compatible Host: https://ACCOUNT_ID.r2.cloudflarestorage.com
# Region: auto
# Bucket: portainer-backups
# Access Key: your-r2-access-key-id
# Secret Key: your-r2-secret-access-key
```

## Step 6: Set a Backup Schedule

The schedule uses standard cron syntax:

```
# Examples:
"0 2 * * *"     # Daily at 2 AM
"0 */12 * * *"  # Every 12 hours
"0 2 * * 0"     # Weekly on Sunday at 2 AM
"0 2 1 * *"     # Monthly on the 1st at 2 AM
```

## Step 7: Verify Backups Are Running

```bash
# Check S3 for backup files
aws s3 ls s3://my-portainer-backups/portainer/ --recursive

# Expected output:
# 2026-03-20 02:00:15    456789 portainer/portainer-2026-03-20T02:00:00Z.tar.gz

# Check backup file size (should be > 10KB for a configured instance)
aws s3 ls s3://my-portainer-backups/portainer/ \
  --recursive --human-readable --summarize
```

## Step 8: Test Restore from S3

Verify you can actually restore before you need to:

```bash
# Download a backup
aws s3 cp \
  s3://my-portainer-backups/portainer/portainer-2026-03-20T02:00:00Z.tar.gz \
  /tmp/portainer-backup-test.tar.gz

# Restore to a test instance
docker stop portainer-test 2>/dev/null; docker rm portainer-test 2>/dev/null
docker volume rm portainer_data_test 2>/dev/null || true

docker run --rm \
  -v portainer_data_test:/data \
  -v /tmp:/backup \
  alpine tar xzf /backup/portainer-backup-test.tar.gz -C /data

# The backup is encrypted — restore via Portainer's restore UI
# Settings → Backup → Restore → Upload the downloaded file
# Enter the encryption password
```

## Step 9: Monitor Backup Status

```bash
# Check Portainer logs for backup activity
docker logs portainer 2>&1 | grep -i "backup\|s3" | tail -10

# Set up alerting if backup fails
# Via Portainer notification settings or external monitoring
```

## Conclusion

Portainer Business Edition's S3 backup integration provides automated, encrypted backups with zero operational overhead. Configure it once, verify it works by testing a restore, and then forget about it — your Portainer configuration is protected. Use the lifecycle policy on your S3 bucket to automatically age out old backups and control storage costs.
