# How to Create S3 Buckets with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, S3, Terraform, IaC, DevOps

Description: Learn how to create and configure AWS S3 buckets with OpenTofu, including versioning, encryption, lifecycle rules, and access policies.

## Introduction

S3 bucket configuration in OpenTofu spans multiple resources. Modern AWS provider versions (4.x+) split bucket settings into separate resources (`aws_s3_bucket_versioning`, `aws_s3_bucket_server_side_encryption_configuration`, etc.) instead of inline arguments. This guide covers creating a fully configured S3 bucket with all production-ready settings.

## Basic S3 Bucket

```hcl
resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name

  tags = {
    Name        = var.bucket_name
    Environment = var.environment
  }
}
```

## Blocking Public Access

Always block public access unless the bucket intentionally hosts public content:

```hcl
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## Enabling Versioning

```hcl
resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

## Server-Side Encryption

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.s3.arn
    }
    bucket_key_enabled = true  # Reduces KMS API costs
  }
}
```

## Lifecycle Rules

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

    filter {
      prefix = "data/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    expiration {
      days = 365
    }
  }

  rule {
    id     = "cleanup-old-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 30
    }

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

## Bucket Policy

```hcl
data "aws_iam_policy_document" "bucket_policy" {
  # Enforce HTTPS
  statement {
    sid    = "DenyHTTP"
    effect = "Deny"
    principals {
      type        = "*"
      identifiers = ["*"]
    }
    actions   = ["s3:*"]
    resources = [
      aws_s3_bucket.main.arn,
      "${aws_s3_bucket.main.arn}/*"
    ]
    condition {
      test     = "Bool"
      variable = "aws:SecureTransport"
      values   = ["false"]
    }
  }

  # Allow specific IAM role
  statement {
    sid    = "AllowAppRole"
    effect = "Allow"
    principals {
      type        = "AWS"
      identifiers = [aws_iam_role.app.arn]
    }
    actions   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
    resources = ["${aws_s3_bucket.main.arn}/*"]
  }
}

resource "aws_s3_bucket_policy" "main" {
  bucket = aws_s3_bucket.main.id
  policy = data.aws_iam_policy_document.bucket_policy.json

  # Ensure public access block is configured before applying policy
  depends_on = [aws_s3_bucket_public_access_block.main]
}
```

## CORS Configuration (For Web Apps)

```hcl
resource "aws_s3_bucket_cors_configuration" "main" {
  bucket = aws_s3_bucket.main.id

  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["https://${var.domain_name}"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}
```

## Cross-Region Replication

```hcl
resource "aws_s3_bucket_replication_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.replica.arn
      storage_class = "STANDARD_IA"
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }

  depends_on = [aws_s3_bucket_versioning.main]
}
```

## Outputs

```hcl
output "bucket_name"   { value = aws_s3_bucket.main.bucket }
output "bucket_arn"    { value = aws_s3_bucket.main.arn }
output "bucket_domain" { value = aws_s3_bucket.main.bucket_regional_domain_name }
```

## Conclusion

A production S3 bucket in OpenTofu requires configuring multiple companion resources: public access block, versioning, encryption, lifecycle rules, and bucket policy. The modern AWS provider uses separate resources for each of these settings. Always enable public access blocking, encryption, and HTTPS enforcement. Use lifecycle rules to manage costs by transitioning old data to cheaper storage classes. Consider cross-region replication for disaster recovery of critical data.
