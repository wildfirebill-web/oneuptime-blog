# How to Set Up S3 Object Lock with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, S3, Object Lock, WORM, Compliance, Infrastructure as Code

Description: Learn how to configure S3 Object Lock with COMPLIANCE and GOVERNANCE modes using OpenTofu to create WORM (Write Once Read Many) storage for regulatory compliance requirements.

## Introduction

S3 Object Lock prevents objects from being deleted or overwritten for a fixed period or indefinitely. It implements WORM (Write Once Read Many) storage, which is required by regulations like SEC 17a-4(f), CFTC, and HIPAA. Object Lock can be configured at the bucket level or per object.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with S3 permissions
- Note: Object Lock must be enabled at bucket creation—it cannot be added to an existing bucket

## Step 1: Create a Bucket with Object Lock Enabled

```hcl
# Object Lock must be enabled at bucket creation
# It cannot be enabled on existing buckets
resource "aws_s3_bucket" "worm" {
  bucket = var.compliance_bucket_name

  # Object Lock requires versioning - versioning is automatically enabled
  object_lock_enabled = true

  tags = {
    Name        = var.compliance_bucket_name
    Purpose     = "Compliance"
    ObjectLock  = "enabled"
  }
}
```

## Step 2: Configure Default Object Lock Retention

```hcl
# Set a default retention policy for all objects in the bucket
resource "aws_s3_bucket_object_lock_configuration" "worm" {
  bucket = aws_s3_bucket.worm.id

  rule {
    default_retention {
      # COMPLIANCE mode: Even root account cannot delete until period expires
      # GOVERNANCE mode: Users with s3:BypassGovernanceRetention can override
      mode = "COMPLIANCE"
      days = 365  # 1-year retention (or use "years" for longer periods)
    }
  }
}

# Encryption for the WORM bucket
resource "aws_s3_bucket_server_side_encryption_configuration" "worm" {
  bucket = aws_s3_bucket.worm.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = var.kms_key_arn
    }
    bucket_key_enabled = true
  }
}
```

## Step 3: Apply Object Lock on Individual Objects

```hcl
# Upload a compliance record with per-object retention override
resource "aws_s3_object" "financial_record" {
  bucket = aws_s3_bucket.worm.id
  key    = "financial/2026/q1-report.pdf"
  source = "${path.module}/reports/q1-report.pdf"

  # Override the default retention with a longer period for this object
  object_lock_mode              = "COMPLIANCE"
  object_lock_retain_until_date = "2033-01-01T00:00:00Z"  # 7-year retention

  # Apply a legal hold in addition to retention
  object_lock_legal_hold_status = "ON"

  tags = {
    RecordType = "FinancialReport"
    Quarter    = "Q1-2026"
  }
}
```

## Step 4: GOVERNANCE Mode for Testing

```hcl
# GOVERNANCE mode allows authorized users to bypass retention
resource "aws_s3_bucket" "governance" {
  bucket              = "${var.compliance_bucket_name}-governance"
  object_lock_enabled = true

  tags = { Purpose = "Governance", ObjectLock = "governance-mode" }
}

resource "aws_s3_bucket_object_lock_configuration" "governance" {
  bucket = aws_s3_bucket.governance.id

  rule {
    default_retention {
      mode  = "GOVERNANCE"
      years = 3
    }
  }
}

# IAM policy to allow bypassing GOVERNANCE mode
resource "aws_iam_policy" "bypass_governance" {
  name        = "S3GovernanceBypassPolicy"
  description = "Allow bypassing Object Lock GOVERNANCE mode"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "s3:BypassGovernanceRetention"
      Resource = "${aws_s3_bucket.governance.arn}/*"
    }]
  })
}
```

## Step 5: Verify Object Lock Settings

```bash
# Check the lock status of an object
aws s3api head-object \
  --bucket my-compliance-bucket \
  --key financial/2026/q1-report.pdf \
  --query '{LockMode: ObjectLockMode, RetainUntil: ObjectLockRetainUntilDate}'

# Check bucket-level Object Lock configuration
aws s3api get-object-lock-configuration \
  --bucket my-compliance-bucket
```

## Step 6: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

S3 Object Lock provides immutable storage for regulatory compliance where records must be preserved unchanged for defined periods. Use COMPLIANCE mode for strict regulatory requirements where even AWS cannot override retention, and GOVERNANCE mode for operational scenarios where authorized administrators may need to intervene. Always combine Object Lock with bucket versioning, encryption, and access logging for a complete compliance posture.
