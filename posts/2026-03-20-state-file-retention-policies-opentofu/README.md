# How to Implement State File Retention Policies in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, State Management, Retention, Lifecycle, S3

Description: Learn how to implement retention policies for OpenTofu state files to balance disaster recovery capabilities with storage costs using S3 lifecycle rules.

## Introduction

State file retention policies define how long historical versions of state files are kept. Keeping too many versions wastes storage; keeping too few limits your recovery window. Well-designed retention policies use tiered storage to balance cost and recoverability.

## Tiered Retention Policy

```hcl
# Keep recent versions in S3 Standard, older versions in cheaper storage
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  bucket = aws_s3_bucket.state.id

  rule {
    id     = "state-version-retention"
    status = "Enabled"

    # Target non-current (historical) versions
    noncurrent_version_transition {
      # Move to Standard-IA after 30 days (versions older than 30 days)
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_transition {
      # Move to Glacier after 90 days for long-term retention
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      # Delete versions older than 365 days
      noncurrent_days = 365
    }
  }

  # Also handle incomplete multipart uploads
  rule {
    id     = "abort-incomplete-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

## Environment-Specific Retention

Production state deserves longer retention than development:

```hcl
locals {
  retention_policies = {
    dev = {
      standard_ia_days   = 7
      glacier_days       = 30
      expiration_days    = 90
    }
    staging = {
      standard_ia_days   = 14
      glacier_days       = 60
      expiration_days    = 180
    }
    prod = {
      standard_ia_days   = 30
      glacier_days       = 90
      expiration_days    = 365
    }
  }
}

resource "aws_s3_bucket_lifecycle_configuration" "state" {
  for_each = local.retention_policies

  bucket = aws_s3_bucket.state.id

  rule {
    id     = "${each.key}-retention"
    status = "Enabled"

    filter {
      prefix = "${each.key}/"  # Apply to env-specific prefix
    }

    noncurrent_version_transition {
      noncurrent_days = each.value.standard_ia_days
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_transition {
      noncurrent_days = each.value.glacier_days
      storage_class   = "GLACIER"
    }

    noncurrent_version_expiration {
      noncurrent_days = each.value.expiration_days
    }
  }
}
```

## Compliance-Driven Retention

For regulated workloads, retain state for auditing purposes:

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "compliance" {
  bucket = aws_s3_bucket.state_compliance.id

  rule {
    id     = "compliance-retention"
    status = "Enabled"

    # For SOC2/HIPAA: keep 7 years of state history
    noncurrent_version_transition {
      noncurrent_days = 90
      storage_class   = "GLACIER"
    }

    noncurrent_version_transition {
      noncurrent_days = 180
      storage_class   = "DEEP_ARCHIVE"
    }

    # Never auto-expire - handle deletion manually after audit period
    # noncurrent_version_expiration { noncurrent_days = 2555 }
  }
}

# Object lock for immutable compliance records
resource "aws_s3_bucket_object_lock_configuration" "compliance" {
  bucket = aws_s3_bucket.state_compliance.id

  rule {
    default_retention {
      mode  = "COMPLIANCE"
      years = 7
    }
  }
}
```

## Cost Estimation for Retention Policies

```bash
# Estimate storage costs for current state versions
aws s3api list-object-versions \
  --bucket my-company-tofu-state \
  --query 'sum(Versions[].Size)' \
  --output text

# Count non-current versions accumulating
aws s3api list-object-versions \
  --bucket my-company-tofu-state \
  --query 'length(Versions[?IsLatest==`false`])' \
  --output text
```

## Conclusion

Implement tiered retention policies that match the recovery needs of each environment. Development environments rarely need more than 90 days of history, while production deserves at least a year. For compliance workloads, use S3 Object Lock to prevent accidental deletion of state history required for audits. Review and adjust policies annually as storage costs and compliance requirements evolve.
