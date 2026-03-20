# How to Create DynamoDB Global Tables with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, DynamoDB, Global Tables, Multi-Region, Replication, Infrastructure as Code

Description: Learn how to configure DynamoDB Global Tables with OpenTofu to enable multi-region active-active replication for low-latency access and disaster recovery.

## Introduction

DynamoDB Global Tables provides a fully managed, multi-region, multi-active database solution. Each region can accept both reads and writes, and DynamoDB automatically propagates changes across all replicas with sub-second latency. Conflict resolution uses a last-writer-wins approach based on timestamps.

## Prerequisites

- OpenTofu v1.6+
- AWS credentials with DynamoDB permissions in all target regions
- DynamoDB tables must use `PAY_PER_REQUEST` or provisioned capacity with auto scaling

## Step 1: Create Global Table with Replicas

```hcl
# DynamoDB Global Table with replicas in multiple regions
resource "aws_dynamodb_table" "global" {
  name             = "${var.project_name}-global-table"
  billing_mode     = "PAY_PER_REQUEST"
  hash_key         = "pk"
  range_key        = "sk"
  stream_enabled   = true
  stream_view_type = "NEW_AND_OLD_IMAGES"  # Required for global tables

  attribute {
    name = "pk"
    type = "S"
  }

  attribute {
    name = "sk"
    type = "S"
  }

  attribute {
    name = "gsi1pk"
    type = "S"
  }

  global_secondary_index {
    name            = "GSI1"
    hash_key        = "gsi1pk"
    range_key       = "sk"
    projection_type = "ALL"
  }

  server_side_encryption {
    enabled     = true
    kms_key_arn = var.kms_key_arn
  }

  point_in_time_recovery {
    enabled = true
  }

  # Add replicas to additional regions
  replica {
    region_name = "us-west-2"
    kms_key_arn = var.us_west_2_kms_key_arn
    point_in_time_recovery = true
  }

  replica {
    region_name = "eu-west-1"
    kms_key_arn = var.eu_west_1_kms_key_arn
    point_in_time_recovery = true
  }

  tags = {
    Name        = "${var.project_name}-global-table"
    Environment = var.environment
  }
}
```

## Step 2: IAM Policy for Multi-Region Access

```hcl
resource "aws_iam_policy" "dynamodb_global" {
  name = "${var.project_name}-dynamodb-global-access"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan",
          "dynamodb:BatchGetItem",
          "dynamodb:BatchWriteItem"
        ]
        Resource = [
          aws_dynamodb_table.global.arn,
          "${aws_dynamodb_table.global.arn}/index/*"
        ]
      }
    ]
  })
}
```

## Step 3: CloudWatch Alarms for Replication Latency

```hcl
resource "aws_cloudwatch_metric_alarm" "replication_latency" {
  alarm_name          = "${var.project_name}-dynamodb-replication-latency"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "ReplicationLatency"
  namespace           = "AWS/DynamoDB"
  period              = 60
  statistic           = "Maximum"
  threshold           = 500  # Alert if replication lag > 500ms

  dimensions = {
    TableName   = aws_dynamodb_table.global.name
    ReceivingRegion = "eu-west-1"
  }

  alarm_actions = [var.sns_topic_arn]
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# Verify replicas
aws dynamodb describe-table \
  --table-name my-project-global-table \
  --query 'Table.Replicas'
```

## Conclusion

DynamoDB Global Tables enable active-active multi-region architectures without custom replication code. Direct traffic to the nearest regional endpoint using Route 53 latency-based routing, and design your application to handle eventual consistency—reads from non-writer regions may briefly lag behind. Use separate KMS keys per region for encryption, and ensure all replicas have the same GSI configuration which is enforced automatically by DynamoDB.
