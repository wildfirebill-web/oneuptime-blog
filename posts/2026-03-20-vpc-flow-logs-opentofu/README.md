# How to Configure VPC Flow Logs with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, VPC Flow Logs, AWS, Security, Monitoring, Infrastructure as Code

Description: Learn how to enable and configure AWS VPC Flow Logs using OpenTofu — publishing to CloudWatch Logs or S3, customizing log format, and querying logs with Athena.

## Introduction

VPC Flow Logs capture IP traffic information for network interfaces in your VPC. They're essential for security investigation, network troubleshooting, and compliance. OpenTofu manages flow log configuration, IAM roles, and log destinations.

## IAM Role for Flow Logs to CloudWatch

```hcl
resource "aws_iam_role" "flow_logs" {
  name = "${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "flow-logs-cloudwatch-policy"
  role = aws_iam_role.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ]
      Resource = "*"
    }]
  })
}
```

## CloudWatch Logs Destination

```hcl
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flow-logs/${var.environment}"
  retention_in_days = 90

  tags = { Environment = var.environment }
}

resource "aws_flow_log" "vpc_cloudwatch" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"  # ALL, ACCEPT, or REJECT
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  # Custom log format (default is limited)
  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${start} $${end} $${action} $${log-status} $${vpc-id} $${subnet-id} $${instance-id} $${tcp-flags} $${type} $${pkt-srcaddr} $${pkt-dstaddr}"

  tags = {
    Name        = "${var.environment}-vpc-flow-logs"
    Environment = var.environment
  }
}
```

## S3 Destination (for Athena Queries)

```hcl
resource "aws_s3_bucket" "flow_logs" {
  bucket = "${var.project}-vpc-flow-logs-${var.environment}"
}

resource "aws_s3_bucket_lifecycle_configuration" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"
    expiration { days = 365 }
    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }
}

resource "aws_s3_bucket_policy" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Sid       = "AWSLogDeliveryWrite"
      Effect    = "Allow"
      Principal = { Service = "delivery.logs.amazonaws.com" }
      Action    = "s3:PutObject"
      Resource  = "${aws_s3_bucket.flow_logs.arn}/AWSLogs/${data.aws_caller_identity.current.account_id}/*"
      Condition = {
        StringEquals = { "s3:x-amz-acl" = "bucket-owner-full-control" }
      }
    }, {
      Sid       = "AWSLogDeliveryAclCheck"
      Effect    = "Allow"
      Principal = { Service = "delivery.logs.amazonaws.com" }
      Action    = "s3:GetBucketAcl"
      Resource  = aws_s3_bucket.flow_logs.arn
    }]
  })
}

resource "aws_flow_log" "vpc_s3" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  log_destination_type = "s3"
  log_destination      = "${aws_s3_bucket.flow_logs.arn}/vpc-flow-logs/"

  log_format = "$${version} $${account-id} $${vpc-id} $${subnet-id} $${instance-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${action} $${log-status}"

  # Parquet format for efficient Athena queries
  destination_options {
    file_format                = "parquet"
    hive_compatible_partitions = true
    per_hour_partition         = true
  }

  tags = { Environment = var.environment }
}
```

## Athena Table for Querying

```hcl
resource "aws_glue_catalog_table" "flow_logs" {
  name          = "vpc_flow_logs"
  database_name = "vpc_logs"
  table_type    = "EXTERNAL_TABLE"

  parameters = {
    EXTERNAL             = "TRUE"
    "classification"     = "parquet"
    "projection.enabled" = "true"
    "projection.date.type" = "date"
    "projection.date.range" = "2024/01/01,NOW"
    "projection.date.format" = "yyyy/MM/dd"
    "projection.date.interval" = "1"
    "projection.date.interval.unit" = "DAYS"
    "storage.location.template" = "s3://${aws_s3_bucket.flow_logs.bucket}/vpc-flow-logs/$${date}"
  }

  storage_descriptor {
    location      = "s3://${aws_s3_bucket.flow_logs.bucket}/vpc-flow-logs/"
    input_format  = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat"
    output_format = "org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat"

    columns {
      name = "version"
      type = "int"
    }
    columns {
      name = "srcaddr"
      type = "string"
    }
    columns {
      name = "dstaddr"
      type = "string"
    }
    columns {
      name = "srcport"
      type = "int"
    }
    columns {
      name = "dstport"
      type = "int"
    }
    columns {
      name = "action"
      type = "string"
    }
  }
}
```

## Conclusion

VPC Flow Logs with OpenTofu capture all network traffic for security and compliance. Use CloudWatch Logs for real-time alerting on rejected connections, and S3 with Parquet format for efficient historical analysis via Athena. Enable flow logs on all VPCs in production — the cost is primarily storage, which Glacier transitions and lifecycle rules keep manageable. Custom log formats provide richer data including VPC ID, subnet ID, and instance ID for easier troubleshooting.
