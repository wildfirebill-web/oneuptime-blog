# How to Set Up AWS CloudTrail with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, CloudTrail, Audit Logging, Security

Description: Learn how to configure AWS CloudTrail with OpenTofu to capture API activity across your AWS account and store logs in S3 with encryption and integrity validation.

## Introduction

AWS CloudTrail records API calls made to your AWS account, providing an audit trail for security analysis, compliance, and troubleshooting. This guide sets up a multi-region trail with S3 log delivery, KMS encryption, and log file validation.

## S3 Bucket for Trail Logs

CloudTrail needs a bucket with a specific bucket policy allowing it to deliver logs:

```hcl
resource "aws_s3_bucket" "cloudtrail" {
  bucket = "${var.account_id}-cloudtrail-logs"
  tags   = { Name = "cloudtrail-logs" }
}

resource "aws_s3_bucket_public_access_block" "cloudtrail" {
  bucket                  = aws_s3_bucket.cloudtrail.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

data "aws_iam_policy_document" "cloudtrail_bucket" {
  statement {
    sid       = "AWSCloudTrailAclCheck"
    effect    = "Allow"
    principals { type = "Service"; identifiers = ["cloudtrail.amazonaws.com"] }
    actions   = ["s3:GetBucketAcl"]
    resources = [aws_s3_bucket.cloudtrail.arn]
  }

  statement {
    sid       = "AWSCloudTrailWrite"
    effect    = "Allow"
    principals { type = "Service"; identifiers = ["cloudtrail.amazonaws.com"] }
    actions   = ["s3:PutObject"]
    resources = ["${aws_s3_bucket.cloudtrail.arn}/AWSLogs/${var.account_id}/*"]
    condition {
      test     = "StringEquals"
      variable = "s3:x-amz-acl"
      values   = ["bucket-owner-full-control"]
    }
  }
}

resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket     = aws_s3_bucket.cloudtrail.id
  policy     = data.aws_iam_policy_document.cloudtrail_bucket.json
  depends_on = [aws_s3_bucket_public_access_block.cloudtrail]
}
```

## KMS Key for Encryption

```hcl
resource "aws_kms_key" "cloudtrail" {
  description             = "KMS key for CloudTrail log encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
}

resource "aws_kms_alias" "cloudtrail" {
  name          = "alias/cloudtrail"
  target_key_id = aws_kms_key.cloudtrail.key_id
}
```

## CloudTrail Trail

```hcl
resource "aws_cloudtrail" "main" {
  name                          = "${var.name}-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.cloudtrail.arn

  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_cw.arn

  event_selector {
    read_write_type           = "All"
    include_management_events = true

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::"]
    }
  }

  tags       = { Name = "${var.name}-trail" }
  depends_on = [aws_s3_bucket_policy.cloudtrail]
}

resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/${var.name}"
  retention_in_days = 90
  kms_key_id        = aws_kms_key.cloudtrail.arn
}
```

## Root Account Usage Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "root_usage" {
  alarm_name          = "root-account-usage"
  alarm_description   = "Root account has been used — investigate immediately"
  namespace           = "CloudTrailMetrics"
  metric_name         = "RootAccountUsageEventCount"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  threshold           = 1
  evaluation_periods  = 1
  period              = 300
  statistic           = "Sum"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
  treat_missing_data  = "notBreaching"
}
```

## Conclusion

CloudTrail is non-negotiable for security and compliance. Enable log file validation to detect tampering, use KMS encryption for log confidentiality, and forward logs to CloudWatch Logs for real-time alerting on suspicious activity.
