# How to Schedule Automatic Backups to S3 in Portainer Business Edition (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Business Edition, S3, Backup, Automation, AWS, Data Protection

Description: Learn how to configure Portainer Business Edition's built-in S3 backup scheduling to automatically store encrypted backups in AWS S3 or compatible object storage.

---

Portainer Business Edition includes native S3 backup scheduling. It encrypts the backup and uploads it directly to your S3 bucket on a configurable schedule, eliminating the need for external backup scripts.

## Prerequisites

- Portainer Business Edition with a valid license
- An S3 bucket (AWS S3, MinIO, or any S3-compatible storage)
- IAM credentials with `s3:PutObject` permission

## Configuring S3 Backups

1. Log in to Portainer.
2. Go to **Settings > General**.
3. Scroll to **Backup up Portainer**.
4. Toggle **Schedule automatic backups**.
5. Fill in the S3 configuration.

## Required S3 Settings

| Setting | Description | Example |
|---|---|---|
| Access key ID | AWS/MinIO access key | `AKIAIOSFODNN7EXAMPLE` |
| Secret access key | AWS/MinIO secret key | `wJalrXUtnFEMI/K7MDENG/...` |
| Region | AWS region | `us-east-1` |
| Bucket name | Target S3 bucket | `portainer-backups` |
| Bucket directory | Path prefix within bucket | `portainer/prod/` |
| Password | Encryption password | A strong random string |
| Schedule | Cron expression | `0 2 * * *` (2 AM daily) |

## Setting Up S3 Permissions

Create a minimal IAM policy for Portainer backup access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::portainer-backups",
        "arn:aws:s3:::portainer-backups/portainer/*"
      ]
    }
  ]
}
```

## Using MinIO as S3 Backend

For self-hosted backup storage, use a local MinIO instance:

| Setting | Value |
|---|---|
| S3 Compatible Host | `http://minio:9000` |
| Access Key ID | Your MinIO access key |
| Bucket | `portainer-backups` |

## Cron Schedule Examples

```text
# Every day at 2 AM

0 2 * * *

# Every 6 hours
0 */6 * * *

# Every Sunday at midnight
0 0 * * 0
```

## Restoring from S3 Backup

1. In Portainer go to **Settings > General > Backup up Portainer**.
2. Click **Restore from S3**.
3. Select the backup file from your S3 bucket.
4. Enter the encryption password.
5. Click **Restore**.

Portainer will download the backup, decrypt it, and restore all settings, users, and stack configurations.
