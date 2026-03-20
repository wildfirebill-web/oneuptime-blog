# How to Configure the S3 Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backends, AWS

Description: Learn how to configure the S3 backend in OpenTofu to store state in Amazon S3 with optional DynamoDB locking and encryption.

## Introduction

The S3 backend is the most popular remote backend for AWS users. It stores state in an S3 bucket with optional DynamoDB state locking, versioning, and server-side encryption. OpenTofu includes an enhanced S3 backend with additional features like native state locking without DynamoDB.

## Basic Configuration

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket = "my-tofu-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Creating the S3 Bucket

```hcl
# Create the state bucket with best practices
resource "aws_s3_bucket" "tofu_state" {
  bucket = "my-tofu-state"
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
      sse_algorithm = "AES256"
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

## With DynamoDB Locking

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tofu-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "tofu-state-lock"
    encrypt        = true
  }
}
```

```hcl
# Create the DynamoDB lock table
resource "aws_dynamodb_table" "tofu_lock" {
  name         = "tofu-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

## With KMS Encryption

```hcl
terraform {
  backend "s3" {
    bucket         = "my-tofu-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/abc-123"
    dynamodb_table = "tofu-state-lock"
  }
}
```

## IAM Permissions Required

```hcl
resource "aws_iam_policy" "tofu_backend" {
  name = "opentofu-backend-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
        Resource = "arn:aws:s3:::my-tofu-state/production/*"
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ListBucket"]
        Resource = "arn:aws:s3:::my-tofu-state"
      },
      {
        Effect   = "Allow"
        Action   = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:DeleteItem"
        ]
        Resource = "arn:aws:dynamodb:us-east-1:*:table/tofu-state-lock"
      }
    ]
  })
}
```

## Multiple Environments

Separate state per environment using different keys:

```hcl
# production/backend.tf
terraform {
  backend "s3" {
    bucket = "my-tofu-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}

# staging/backend.tf
terraform {
  backend "s3" {
    bucket = "my-tofu-state"
    key    = "staging/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Initializing the Backend

```bash
tofu init

# Output:
# Initializing the backend...
# Successfully configured the backend "s3"!
```

## Conclusion

The S3 backend provides reliable, scalable remote state storage for AWS environments. Enable versioning for state history, DynamoDB locking for concurrency protection, and KMS encryption for data-at-rest security. Use distinct keys per environment to keep state files isolated while sharing the same bucket.
