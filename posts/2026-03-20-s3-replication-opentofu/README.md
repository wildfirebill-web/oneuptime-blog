# How to Configure S3 Replication with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, S3, Replication, AWS, Disaster Recovery, Infrastructure as Code

Description: Learn how to configure S3 Cross-Region Replication (CRR) and Same-Region Replication (SRR) with OpenTofu — including IAM roles, replication rules, filter-based replication, and replica modification sync.

## Introduction

S3 Replication copies objects automatically between buckets — across regions (CRR) for disaster recovery, or within the same region (SRR) for log aggregation and compliance. OpenTofu manages the source bucket's replication configuration, the destination bucket, and the IAM role that S3 assumes to perform replication.

## IAM Role for Replication

```hcl
resource "aws_iam_role" "replication" {
  name = "${var.environment}-s3-replication-role"

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
  name = "s3-replication-policy"
  role = aws_iam_role.replication.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetReplicationConfiguration",
          "s3:ListBucket"
        ]
        Resource = [aws_s3_bucket.source.arn]
      },
      {
        Effect = "Allow"
        Action = [
          "s3:GetObjectVersionForReplication",
          "s3:GetObjectVersionAcl",
          "s3:GetObjectVersionTagging"
        ]
        Resource = ["${aws_s3_bucket.source.arn}/*"]
      },
      {
        Effect = "Allow"
        Action = [
          "s3:ReplicateObject",
          "s3:ReplicateDelete",
          "s3:ReplicateTags"
        ]
        Resource = ["${aws_s3_bucket.destination.arn}/*"]
      }
    ]
  })
}
```

## Source and Destination Buckets

```hcl
# Source bucket (us-east-1)
resource "aws_s3_bucket" "source" {
  bucket = "${var.project}-source-${var.environment}"
}

resource "aws_s3_bucket_versioning" "source" {
  bucket = aws_s3_bucket.source.id
  versioning_configuration { status = "Enabled" }
}

# Destination bucket (eu-west-1) — requires separate provider
resource "aws_s3_bucket" "destination" {
  provider = aws.eu_west_1
  bucket   = "${var.project}-replica-${var.environment}"
}

resource "aws_s3_bucket_versioning" "destination" {
  provider = aws.eu_west_1
  bucket   = aws_s3_bucket.destination.id
  versioning_configuration { status = "Enabled" }
}
```

## Cross-Region Replication Configuration

```hcl
resource "aws_s3_bucket_replication_configuration" "crr" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  depends_on = [aws_s3_bucket_versioning.source]

  rule {
    id     = "replicate-all"
    status = "Enabled"

    filter {}  # Empty filter = replicate all objects

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"  # Cost-optimize replicas

      # Replicate encrypted objects
      encryption_configuration {
        replica_kms_key_id = aws_kms_key.replica.arn
      }

      # Sync replica object modifications (tags, ACLs)
      replica_modifications {
        status = "Enabled"
      }

      # Replicate object metadata (delete markers)
      replication_time {
        status = "Enabled"
        time { minutes = 15 }  # SLA for replication
      }

      metrics {
        status = "Enabled"
        event_threshold { minutes = 15 }
      }
    }

    delete_marker_replication {
      status = "Enabled"  # Replicate deletions to replica
    }
  }
}
```

## Prefix-Filtered Replication

```hcl
resource "aws_s3_bucket_replication_configuration" "prefix_crr" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  depends_on = [aws_s3_bucket_versioning.source]

  # Only replicate compliance records
  rule {
    id     = "replicate-compliance"
    status = "Enabled"
    priority = 10

    filter {
      and {
        prefix = "compliance/"
        tags = {
          Replicate = "true"
        }
      }
    }

    destination {
      bucket        = aws_s3_bucket.compliance_replica.arn
      storage_class = "GLACIER"
    }

    delete_marker_replication {
      status = "Disabled"  # Don't replicate deletions for compliance data
    }
  }

  # Replicate logs separately
  rule {
    id       = "replicate-logs"
    status   = "Enabled"
    priority = 20

    filter {
      prefix = "logs/"
    }

    destination {
      bucket        = aws_s3_bucket.log_replica.arn
      storage_class = "STANDARD_IA"
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

## Same-Region Replication (SRR) for Log Aggregation

```hcl
# SRR: Copy logs from multiple account buckets into a central log bucket
resource "aws_s3_bucket_replication_configuration" "srr" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.source.id

  depends_on = [aws_s3_bucket_versioning.source]

  rule {
    id     = "srr-log-aggregation"
    status = "Enabled"

    filter {}

    destination {
      bucket        = aws_s3_bucket.central_logs.arn
      storage_class = "STANDARD"
      account       = var.log_account_id  # Cross-account SRR

      access_control_translation {
        owner = "Destination"
      }
    }
  }
}
```

## Replication Metrics Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "replication_latency" {
  alarm_name          = "${var.environment}-s3-replication-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ReplicationLatency"
  namespace           = "AWS/S3"
  period              = 300
  statistic           = "Maximum"
  threshold           = 900  # 15 minutes
  alarm_description   = "S3 replication latency exceeded 15 minutes"

  dimensions = {
    SourceBucket      = aws_s3_bucket.source.id
    DestinationBucket = aws_s3_bucket.destination.id
    RuleId            = "replicate-all"
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Conclusion

S3 Replication with OpenTofu provides automated data redundancy without manual copy jobs. Enable versioning on both source and destination — it's required for replication. Use S3 Replication Time Control (RTC) for a 15-minute SLA when your RPO requires guaranteed replication timing. Set `delete_marker_replication = "Disabled"` for compliance archives to prevent accidental deletion propagation. Monitor `ReplicationLatency` and `BytesPendingReplication` CloudWatch metrics to detect replication lag before it impacts your RPO.
