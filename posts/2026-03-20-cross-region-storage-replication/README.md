# How to Configure Cross-Region Storage Replication with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, S3, Replication, Cross-Region, Infrastructure as Code

Description: Learn how to set up AWS S3 cross-region replication using OpenTofu to ensure data durability and disaster recovery across regions.

---

S3 Cross-Region Replication (CRR) automatically copies objects from a source bucket in one AWS region to a destination bucket in another. OpenTofu manages the replication configuration, IAM roles, and bucket policies.

---

## Create Source and Destination Buckets

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

resource "aws_s3_bucket" "source" {
  provider = aws.us_east
  bucket   = "my-app-data-source"
}

resource "aws_s3_bucket_versioning" "source" {
  provider = aws.us_east
  bucket   = aws_s3_bucket.source.id
  versioning_configuration {
    status = "Enabled"  # Required for replication
  }
}

resource "aws_s3_bucket" "destination" {
  provider = aws.eu_west
  bucket   = "my-app-data-replica"
}

resource "aws_s3_bucket_versioning" "destination" {
  provider = aws.eu_west
  bucket   = aws_s3_bucket.destination.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

---

## Create the Replication IAM Role

```hcl
resource "aws_iam_role" "replication" {
  name = "s3-crr-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "s3.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "replication" {
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = ["s3:GetReplicationConfiguration", "s3:ListBucket"]
        Resource = aws_s3_bucket.source.arn
      },
      {
        Effect   = "Allow"
        Action   = ["s3:GetObjectVersionForReplication", "s3:GetObjectVersionAcl"]
        Resource = "${aws_s3_bucket.source.arn}/*"
      },
      {
        Effect   = "Allow"
        Action   = ["s3:ReplicateObject", "s3:ReplicateDelete"]
        Resource = "${aws_s3_bucket.destination.arn}/*"
      }
    ]
  })
}
```

---

## Configure Replication on the Source Bucket

```hcl
resource "aws_s3_bucket_replication_configuration" "main" {
  provider   = aws.us_east
  role       = aws_iam_role.replication.arn
  bucket     = aws_s3_bucket.source.id
  depends_on = [aws_s3_bucket_versioning.source]

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

---

## Verify Replication

```bash
# Upload a test object

aws s3 cp test.txt s3://my-app-data-source/ --region us-east-1

# Check replication status
aws s3api head-object   --bucket my-app-data-source   --key test.txt   --query 'ReplicationStatus'
# "COMPLETED"

# Verify in destination region
aws s3 ls s3://my-app-data-replica/ --region eu-west-1
```

---

## Summary

Enable versioning on both source and destination buckets, create an IAM role with replication permissions, and configure `aws_s3_bucket_replication_configuration` with a rule pointing to the destination bucket ARN. Use multiple providers with different regions to manage cross-region resources in a single OpenTofu configuration.
