# How to Configure S3 Versioning with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, S3, Versioning, AWS, Data Protection, Infrastructure as Code

Description: Learn how to configure S3 bucket versioning with OpenTofu - enabling versioning, managing MFA delete, combining versioning with lifecycle rules for cost control, and restoring deleted objects.

## Introduction

S3 Versioning preserves every version of every object written to a bucket, protecting against accidental deletion and overwrites. OpenTofu manages versioning state, MFA delete protection, Object Lock for compliance, and the lifecycle rules needed to control version storage costs.

## Enable Versioning

```hcl
resource "aws_s3_bucket" "app" {
  bucket = "${var.project}-app-${var.environment}"
  tags   = { Environment = var.environment }
}

resource "aws_s3_bucket_versioning" "app" {
  bucket = aws_s3_bucket.app.id

  versioning_configuration {
    status = "Enabled"  # or "Suspended" to pause without losing existing versions
  }
}
```

## MFA Delete Protection

```hcl
# MFA delete requires the bucket owner's MFA device for permanent deletion

# Must be configured via CLI or SDK - OpenTofu can represent the desired state
resource "aws_s3_bucket_versioning" "critical" {
  bucket = aws_s3_bucket.critical.id

  versioning_configuration {
    status     = "Enabled"
    mfa_delete = "Enabled"  # Requires MFA for permanent object deletion
  }

  # Note: Enabling MFA delete requires aws CLI with MFA credentials:
  # aws s3api put-bucket-versioning \
  #   --bucket <bucket> \
  #   --versioning-configuration Status=Enabled,MFADelete=Enabled \
  #   --mfa "arn:aws:iam::ACCOUNT:mfa/device TOTP_CODE"
}
```

## Object Lock for Compliance (Write-Once)

```hcl
resource "aws_s3_bucket" "compliance" {
  bucket        = "${var.project}-compliance-${var.environment}"
  object_lock_enabled = true  # Must be set at creation time
}

resource "aws_s3_bucket_versioning" "compliance" {
  bucket = aws_s3_bucket.compliance.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_object_lock_configuration" "compliance" {
  bucket = aws_s3_bucket.compliance.id

  rule {
    default_retention {
      mode  = "COMPLIANCE"  # or "GOVERNANCE" (admin can override)
      years = 7             # Objects cannot be deleted for 7 years
    }
  }
}
```

## Lifecycle Rules to Manage Version Costs

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "versioned_app" {
  bucket     = aws_s3_bucket.app.id
  depends_on = [aws_s3_bucket_versioning.app]

  rule {
    id     = "version-management"
    status = "Enabled"

    filter {}

    # Keep only the 10 most recent noncurrent versions
    noncurrent_version_expiration {
      noncurrent_days           = 30
      newer_noncurrent_versions = 10
    }

    # Move noncurrent versions to cheaper storage before deletion
    noncurrent_version_transition {
      noncurrent_days = 7
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }

    # Remove delete markers after all versions are gone
    expiration {
      expired_object_delete_marker = true
    }
  }
}
```

## Bucket Policy to Prevent Disabling Versioning

```hcl
resource "aws_s3_bucket_policy" "prevent_versioning_disable" {
  bucket = aws_s3_bucket.app.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyVersioningSuspend"
        Effect = "Deny"
        Principal = { AWS = "*" }
        Action    = "s3:PutBucketVersioning"
        Resource  = aws_s3_bucket.app.arn
        Condition = {
          StringEquals = {
            "s3:VersionStatus" = "Suspended"
          }
        }
      }
    ]
  })
}
```

## Outputs for Version Management

```hcl
output "bucket_id" {
  value = aws_s3_bucket.app.id
}

output "versioning_status" {
  value = aws_s3_bucket_versioning.app.versioning_configuration[0].status
}
```

## Retrieving Specific Versions

```hcl
# Data source to reference a specific version of an object
data "aws_s3_object" "config" {
  bucket     = aws_s3_bucket.app.id
  key        = "config/app.json"
  version_id = var.config_version_id  # Specific version, or omit for latest
}

# Restore: copy a previous version over the current one
# Done via CLI: aws s3api copy-object \
#   --bucket bucket-name \
#   --copy-source bucket-name/key?versionId=VERSION_ID \
#   --key key
```

## Versioning with Replication

```hcl
# Versioning must be enabled before configuring replication
resource "aws_s3_bucket_versioning" "source" {
  bucket = aws_s3_bucket.source.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_replication_configuration" "crr" {
  bucket     = aws_s3_bucket.source.id
  role       = aws_iam_role.replication.arn
  depends_on = [aws_s3_bucket_versioning.source]

  rule {
    id     = "full-replication"
    status = "Enabled"
    filter {}

    destination {
      bucket = aws_s3_bucket.destination.arn
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

## Conclusion

S3 Versioning with OpenTofu provides a safety net against accidental object deletion and overwrites. Always pair versioning with lifecycle rules to control storage costs - without them, every version of every object accumulates indefinitely. Use Object Lock in COMPLIANCE mode for audit trails that regulators require to be immutable. Enable the `expired_object_delete_marker` expiration rule to clean up orphaned delete markers that accumulate in heavily-updated versioned buckets.
