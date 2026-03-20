# How to Configure S3 Access Logging with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, S3, Access Logging, Audit, Compliance, Infrastructure as Code

Description: Learn how to enable S3 server access logging using OpenTofu to capture detailed records of all requests made to an S3 bucket for auditing, security analysis, and compliance.

## Introduction

S3 server access logging records detailed information about every request made to a bucket, including the requester, bucket name, request time, request action, response status, and error code. This is essential for security audits, compliance requirements, and debugging access issues.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with S3 permissions

## Step 1: Create a Dedicated Logging Bucket

```hcl
# Separate bucket to store access logs
resource "aws_s3_bucket" "logs" {
  bucket = "${var.bucket_name}-access-logs"
  tags   = { Name = "s3-access-logs", Purpose = "Logging" }
}

# Grant S3 log delivery service write access to the logging bucket
resource "aws_s3_bucket_acl" "logs" {
  bucket = aws_s3_bucket.logs.id
  acl    = "log-delivery-write"

  depends_on = [aws_s3_bucket_ownership_controls.logs]
}

resource "aws_s3_bucket_ownership_controls" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    object_ownership = "ObjectWriter"
  }
}

# Apply lifecycle rules to the logging bucket for cost management
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"
    filter {}

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    expiration {
      days = 90  # Keep logs for 90 days
    }
  }
}
```

## Step 2: Enable Logging on the Source Bucket

```hcl
resource "aws_s3_bucket" "source" {
  bucket = var.bucket_name
  tags   = { Name = var.bucket_name }
}

# Enable server access logging
resource "aws_s3_bucket_logging" "source" {
  bucket = aws_s3_bucket.source.id

  # Send logs to the dedicated logging bucket
  target_bucket = aws_s3_bucket.logs.id

  # Prefix to organize logs by source bucket
  target_prefix = "${var.bucket_name}/access-logs/"
}
```

## Step 3: Enable S3 Access Logging via CloudTrail

```hcl
# CloudTrail data events provide more detailed logging than S3 access logs
# and support EventBridge integration for real-time alerting
resource "aws_cloudtrail" "s3_data_events" {
  name                          = "s3-data-events-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail_logs.id
  include_global_service_events = true
  is_multi_region_trail         = true

  event_selector {
    read_write_type           = "All"
    include_management_events = false  # Focus on data events only

    data_resource {
      type   = "AWS::S3::Object"
      # Monitor all objects in the source bucket
      values = ["${aws_s3_bucket.source.arn}/"]
    }
  }

  tags = { Name = "s3-data-events-trail" }
}
```

## Step 4: Create CloudWatch Alarm for Unauthorized Access

```hcl
# Log group for CloudTrail events
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/s3-events"
  retention_in_days = 30
}

# Metric filter for 403 (Access Denied) responses
resource "aws_cloudwatch_log_metric_filter" "s3_403" {
  name           = "s3-access-denied"
  log_group_name = aws_cloudwatch_log_group.cloudtrail.name
  pattern        = "{ ($.eventSource = \"s3.amazonaws.com\") && ($.errorCode = \"AccessDenied\") }"

  metric_transformation {
    name      = "S3AccessDeniedCount"
    namespace = "Security/S3"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "s3_403" {
  alarm_name          = "s3-access-denied-alarm"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "S3AccessDeniedCount"
  namespace           = "Security/S3"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "High rate of S3 access denied errors"
  alarm_actions       = [var.sns_topic_arn]
}
```

## Step 5: Deploy

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

S3 server access logging provides HTTP-level request records while CloudTrail data events provide API-level audit logging with caller identity information. For security monitoring, CloudTrail is more actionable as it captures the IAM principal making each request. Enable both for comprehensive coverage—server access logs for network-level analysis and CloudTrail for security investigations.
