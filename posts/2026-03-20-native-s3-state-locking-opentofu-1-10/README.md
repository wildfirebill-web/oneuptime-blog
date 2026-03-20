# How to Use Native S3 State Locking Introduced in OpenTofu 1.10

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, S3 State Locking, OpenTofu 1.10, Backend, Infrastructure as Code

Description: Learn how to use native S3 state locking introduced in OpenTofu 1.10, which eliminates the need for a separate DynamoDB table to manage state locks.

## Introduction

Before OpenTofu 1.10, using the S3 backend with state locking required a separate DynamoDB table. OpenTofu 1.10 introduced native S3 state locking using S3's conditional write feature, removing the DynamoDB dependency and simplifying your backend configuration.

## Old Approach: S3 + DynamoDB Locking

The traditional setup required two AWS resources for reliable locking.

```hcl
# Old approach - requires a DynamoDB table
terraform {
  backend "s3" {
    bucket         = "my-tofu-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-state-lock"  # extra resource needed
    encrypt        = true
  }
}

# You had to manage this DynamoDB table separately
resource "aws_dynamodb_table" "terraform_lock" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## New Approach: Native S3 Locking (OpenTofu 1.10)

With native S3 locking, no DynamoDB table is required.

```hcl
terraform {
  backend "s3" {
    bucket  = "my-tofu-state"
    key     = "prod/terraform.tfstate"
    region  = "us-east-1"
    encrypt = true

    # Enable native S3 locking - no DynamoDB needed
    use_lockfile = true
  }
}
```

## Setting Up the S3 Bucket with Proper Policies

Create an S3 bucket configured for state storage with native locking.

```hcl
resource "aws_s3_bucket" "tofu_state" {
  bucket = "my-tofu-state-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_versioning" "tofu_state" {
  bucket = aws_s3_bucket.tofu_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "tofu_state" {
  bucket = aws_s3_bucket.tofu_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tofu_state" {
  bucket                  = aws_s3_bucket.tofu_state.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## Required IAM Permissions

The IAM role or user running OpenTofu needs `s3:PutObject` and `s3:DeleteObject` on the lock file.

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
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-tofu-state",
        "arn:aws:s3:::my-tofu-state/*"
      ]
    }
  ]
}
```

## Migrating from DynamoDB Locking

Migrate an existing S3+DynamoDB backend to native S3 locking.

```bash
# 1. Update backend config to use use_lockfile = true
# 2. Remove dynamodb_table from backend config
# 3. Re-initialize the backend
tofu init -reconfigure

# 4. Verify state is accessible
tofu plan

# 5. After confirming everything works, decommission the DynamoDB table
# Note: wait for all teams to migrate before deleting the table
```

## How Native Locking Works

```
Native S3 locking uses S3 conditional writes:

1. tofu apply starts → attempts to write lock file
   s3://bucket/key.tflock (with If-None-Match: * condition)

2. If no existing lock → write succeeds → apply proceeds

3. If lock exists → conditional write fails →
   OpenTofu reports: "Error: state already locked"

4. Apply completes → lock file deleted

This uses S3's atomic conditional write, eliminating
the need for a separate coordination service.
```

## Summary

Native S3 state locking in OpenTofu 1.10 simplifies the backend configuration by eliminating the DynamoDB dependency. New projects no longer need to provision and pay for a DynamoDB table. Existing projects can migrate by adding `use_lockfile = true` and removing `dynamodb_table` from the backend configuration. The native S3 approach provides the same mutual exclusion guarantees with fewer moving parts.
