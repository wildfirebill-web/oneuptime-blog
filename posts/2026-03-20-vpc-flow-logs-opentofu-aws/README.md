# How to Enable VPC Flow Logs with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, VPC, Flow Logs, Monitoring, Infrastructure as Code

Description: Learn how to configure AWS VPC Flow Logs using OpenTofu, including CloudWatch Logs and S3 destinations with the required IAM roles and policies.

## Introduction

Enabling VPC Flow Logs through OpenTofu ensures your network monitoring is consistent across environments and version-controlled. This guide covers both CloudWatch Logs and S3 destinations.

## Flow Logs to CloudWatch Logs

### IAM Role

```hcl
resource "aws_iam_role" "flowlogs" {
  name = "vpc-flowlogs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = { Service = "vpc-flow-logs.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "flowlogs" {
  name = "vpc-flowlogs-policy"
  role = aws_iam_role.flowlogs.id

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

### CloudWatch Log Group

```hcl
resource "aws_cloudwatch_log_group" "flowlogs" {
  name              = "/aws/vpc/flowlogs/${var.vpc_name}"
  retention_in_days = 30

  tags = {
    Environment = var.environment
  }
}
```

### Flow Log Resource

```hcl
resource "aws_flow_log" "vpc" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flowlogs.arn
  log_destination = aws_cloudwatch_log_group.flowlogs.arn

  tags = {
    Name        = "${var.vpc_name}-flow-logs"
    Environment = var.environment
  }
}
```

## Flow Logs to S3

```hcl
resource "aws_s3_bucket" "flowlogs" {
  bucket = "${var.vpc_name}-flow-logs-${data.aws_caller_identity.current.account_id}"
}

resource "aws_s3_bucket_lifecycle_configuration" "flowlogs" {
  bucket = aws_s3_bucket.flowlogs.id

  rule {
    id     = "expire-old-logs"
    status = "Enabled"

    expiration {
      days = 90
    }
  }
}

resource "aws_flow_log" "vpc_s3" {
  vpc_id               = aws_vpc.main.id
  traffic_type         = "ALL"
  log_destination_type = "s3"
  log_destination      = "${aws_s3_bucket.flowlogs.arn}/vpc-logs/"
}
```

## Custom Flow Log Format

Enable more detailed logging with a custom format:

```hcl
resource "aws_flow_log" "detailed" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flowlogs.arn
  log_destination = aws_cloudwatch_log_group.flowlogs.arn

  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${action} $${tcp-flags}"
}
```

## Deploying

```bash
tofu init
tofu plan
tofu apply
```

## Conclusion

OpenTofu makes it easy to enable VPC Flow Logs with the correct IAM permissions and destination configuration as repeatable code. This ensures network monitoring is enabled from day one of VPC creation and configured identically across all environments.
