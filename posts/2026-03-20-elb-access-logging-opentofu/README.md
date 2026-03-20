# How to Configure ELB Access Logging with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ELB, ALB, NLB, Access Logging, Observability, Infrastructure as Code

Description: Learn how to enable access logging for AWS Application Load Balancers and Network Load Balancers using OpenTofu to capture detailed request records for troubleshooting and compliance.

ELB access logs capture detailed request information including client IP, request paths, response codes, latency, and backend instance responses. Managing log configuration in OpenTofu ensures every load balancer in your environment has logging enabled consistently.

## ALB Access Logging

```hcl
# S3 bucket to store ALB access logs
resource "aws_s3_bucket" "alb_logs" {
  bucket = "alb-access-logs-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id

  rule {
    id     = "expire-logs"
    status = "Enabled"
    filter {}

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    expiration {
      days = 90
    }
  }
}

# ALB requires a specific bucket policy to allow ELB service to write logs
data "aws_elb_service_account" "main" {}

resource "aws_s3_bucket_policy" "alb_logs" {
  bucket = aws_s3_bucket.alb_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = { AWS = data.aws_elb_service_account.main.arn }
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.alb_logs.arn}/alb/*"
      },
      {
        Effect    = "Allow"
        Principal = { Service = "delivery.logs.amazonaws.com" }
        Action    = "s3:PutObject"
        Resource  = "${aws_s3_bucket.alb_logs.arn}/alb/*"
        Condition = {
          StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" }
        }
      },
      {
        Effect    = "Allow"
        Principal = { Service = "delivery.logs.amazonaws.com" }
        Action    = "s3:GetBucketAcl"
        Resource  = aws_s3_bucket.alb_logs.arn
      }
    ]
  })
}

# Application Load Balancer with access logging enabled
resource "aws_lb" "main" {
  name               = "production-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "alb"          # Organizes logs under alb/ prefix
    enabled = true
  }

  tags = {
    Environment = var.environment
  }
}
```

## NLB Access Logging

```hcl
# Network Load Balancer with access logging
resource "aws_lb" "nlb" {
  name               = "production-nlb"
  internal           = false
  load_balancer_type = "network"
  subnets            = var.public_subnet_ids

  access_logs {
    bucket  = aws_s3_bucket.alb_logs.id
    prefix  = "nlb"
    enabled = true
  }

  # Enable cross-zone load balancing
  enable_cross_zone_load_balancing = true
}
```

## Athena Table for Log Analysis

```hcl
resource "aws_athena_database" "alb_logs" {
  name   = "alb_access_logs"
  bucket = aws_s3_bucket.alb_logs.id
}

resource "aws_athena_named_query" "create_table" {
  name      = "create-alb-logs-table"
  database  = aws_athena_database.alb_logs.name
  workgroup = "primary"

  query = <<-EOT
    CREATE EXTERNAL TABLE IF NOT EXISTS alb_access_logs (
      type               string,
      time               string,
      elb                string,
      client_ip          string,
      client_port        int,
      target_ip          string,
      target_port        int,
      request_processing_time double,
      target_processing_time  double,
      response_processing_time double,
      elb_status_code    string,
      target_status_code string,
      received_bytes     bigint,
      sent_bytes         bigint,
      request_verb       string,
      request_url        string,
      request_proto      string,
      user_agent         string,
      ssl_cipher         string,
      ssl_protocol       string
    )
    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
    WITH SERDEPROPERTIES (
      'serialization.format' = '1',
      'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) "([^ ]*) (.*) (- |[^ ]*)" "([^"]*)" ([A-Z0-9-_]+) ([A-Za-z0-9.-]*)'
    )
    LOCATION 's3://${aws_s3_bucket.alb_logs.id}/alb/';
  EOT
}
```

## CloudWatch Alarms on 5xx Errors

```hcl
resource "aws_cloudwatch_metric_alarm" "alb_5xx" {
  alarm_name          = "alb-5xx-errors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "HTTPCode_Target_5XX_Count"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Sum"
  threshold           = 10

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
  }

  alarm_description = "ALB backend returning 5xx errors"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}

resource "aws_cloudwatch_metric_alarm" "alb_latency" {
  alarm_name          = "alb-high-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "TargetResponseTime"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  extended_statistic  = "p99"
  threshold           = 1  # 1 second P99 latency threshold

  dimensions = {
    LoadBalancer = aws_lb.main.arn_suffix
  }

  alarm_description = "ALB p99 latency exceeds 1 second"
  alarm_actions     = [aws_sns_topic.alerts.arn]
}
```

## Conclusion

ELB access logging in OpenTofu ensures every load balancer has consistent log collection. Store logs in a dedicated S3 bucket with lifecycle rules to manage cost, use Athena for ad-hoc SQL analysis of access patterns, and combine with CloudWatch metrics alarms for real-time alerting on error rates and latency. For ALBs, always configure the S3 bucket policy to allow the regional ELB service account to write logs.
