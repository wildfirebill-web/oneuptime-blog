# How to Build a Log Aggregation Pipeline with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Log Aggregation, Kinesis, OpenSearch, CloudWatch, Infrastructure as Code

Description: Learn how to build a log aggregation pipeline on AWS with OpenTofu using Kinesis Data Firehose, OpenSearch, and S3 for long-term storage.

## Introduction

A log aggregation pipeline collects logs from multiple sources, processes them, and stores them for search and analysis. On AWS, this typically involves CloudWatch Logs as the source, Kinesis Data Firehose for streaming delivery, OpenSearch Service for search, and S3 for long-term archival. This guide builds the entire pipeline with OpenTofu.

## S3 Bucket for Log Archival

```hcl
resource "aws_s3_bucket" "logs_archive" {
  bucket = "myapp-logs-archive-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs_archive.id

  rule {
    id     = "transition-to-ia"
    status = "Enabled"

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
}
```

## OpenSearch Domain

```hcl
resource "aws_opensearch_domain" "logs" {
  domain_name    = "myapp-logs-${var.environment}"
  engine_version = "OpenSearch_2.11"

  cluster_config {
    instance_type          = var.environment == "prod" ? "r6g.large.search" : "t3.small.search"
    instance_count         = var.environment == "prod" ? 3 : 1
    zone_awareness_enabled = var.environment == "prod"

    dynamic "zone_awareness_config" {
      for_each = var.environment == "prod" ? [1] : []
      content {
        availability_zone_count = 3
      }
    }
  }

  ebs_options {
    ebs_enabled = true
    volume_type = "gp3"
    volume_size = 100
  }

  encrypt_at_rest {
    enabled = true
  }

  node_to_node_encryption {
    enabled = true
  }

  domain_endpoint_options {
    enforce_https       = true
    tls_security_policy = "Policy-Min-TLS-1-2-2019-07"
  }

  advanced_security_options {
    enabled                        = true
    anonymous_auth_enabled         = false
    internal_user_database_enabled = false

    master_user_options {
      master_user_arn = aws_iam_role.opensearch_admin.arn
    }
  }

  lifecycle {
    prevent_destroy = true
  }
}
```

## Kinesis Data Firehose

```hcl
resource "aws_kinesis_firehose_delivery_stream" "app_logs" {
  name        = "myapp-logs-${var.environment}"
  destination = "opensearch"

  opensearch_configuration {
    domain_arn = aws_opensearch_domain.logs.arn
    role_arn   = aws_iam_role.firehose.arn
    index_name = "myapp-logs"
    index_rotation_period = "OneDay"

    s3_backup_mode = "AllDocuments"

    s3_configuration {
      role_arn           = aws_iam_role.firehose.arn
      bucket_arn         = aws_s3_bucket.logs_archive.arn
      prefix             = "logs/year=!{timestamp:yyyy}/month=!{timestamp:MM}/day=!{timestamp:dd}/"
      error_output_prefix = "errors/!{firehose:error-output-type}/"
      buffering_size     = 5
      buffering_interval = 60
      compression_format = "GZIP"
    }

    processing_configuration {
      enabled = true

      processors {
        type = "Lambda"
        parameters {
          parameter_name  = "LambdaArn"
          parameter_value = "${aws_lambda_function.log_processor.arn}:$LATEST"
        }
      }
    }
  }
}
```

## CloudWatch Logs Subscription Filter

```hcl
resource "aws_cloudwatch_log_subscription_filter" "app_logs" {
  name            = "app-logs-to-firehose"
  log_group_name  = aws_cloudwatch_log_group.application.name
  filter_pattern  = ""  # all logs
  destination_arn = aws_kinesis_firehose_delivery_stream.app_logs.arn
  role_arn        = aws_iam_role.cloudwatch_to_firehose.arn
}
```

## Log Processing Lambda

```hcl
resource "aws_lambda_function" "log_processor" {
  function_name = "myapp-log-processor"
  filename      = data.archive_file.log_processor.output_path
  handler       = "index.handler"
  runtime       = "nodejs20.x"
  role          = aws_iam_role.log_processor_lambda.arn
  timeout       = 60

  environment {
    variables = {
      ENVIRONMENT = var.environment
    }
  }
}
```

## Summary

A log aggregation pipeline on AWS flows from application containers → CloudWatch Logs → Kinesis Data Firehose → OpenSearch (for search) and S3 (for archival). Kinesis Firehose handles batching, buffering, and delivery without requiring you to manage servers. Use a Lambda transformation function to parse, enrich, or filter log records before they reach OpenSearch. Configure lifecycle policies on the S3 archive to automatically transition logs from Standard to Glacier storage as they age.
