# How to Configure CloudWatch Cross-Account Observability with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, CloudWatch, Cross-Account, Observability, Multi-Account, Infrastructure as Code

Description: Learn how to configure CloudWatch cross-account observability with OpenTofu to monitor metrics, logs, and traces from multiple AWS accounts in a centralized monitoring account.

## Introduction

CloudWatch cross-account observability enables a centralized monitoring account to view metrics, logs, alarms, and traces from multiple source accounts. Instead of switching between accounts or building custom aggregation pipelines, operators can monitor all workloads from a single CloudWatch console view.

## Prerequisites

- OpenTofu v1.6+
- A monitoring account (sink) and one or more source accounts
- AWS credentials with CloudWatch and Organizations permissions

## Step 1: Set Up the Monitoring Account (Sink)

```hcl
# In the monitoring (central) account
resource "aws_oam_sink" "monitoring" {
  name = "${var.project_name}-monitoring-sink"

  tags = {
    Name = "${var.project_name}-monitoring-sink"
  }
}

# Policy allowing source accounts to link to this sink
resource "aws_oam_sink_policy" "monitoring" {
  sink_identifier = aws_oam_sink.monitoring.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action   = ["oam:CreateLink", "oam:UpdateLink"]
        Effect   = "Allow"
        Resource = "*"
        Principal = {
          AWS = formatlist(
            "arn:aws:iam::%s:root",
            var.source_account_ids  # List of source account IDs
          )
        }
        Condition = {
          ForAllValues:StringEquals = {
            "oam:ResourceTypes" = [
              "AWS::CloudWatch::Metric",
              "AWS::Logs::LogGroup",
              "AWS::XRay::Trace"
            ]
          }
        }
      }
    ]
  })
}

output "sink_arn" {
  value       = aws_oam_sink.monitoring.arn
  description = "Share this ARN with source accounts to create links"
}
```

## Step 2: Link Source Accounts to the Monitoring Sink

```hcl
# In each source account - link to the monitoring sink
resource "aws_oam_link" "to_monitoring" {
  label_template  = "$AccountName"
  resource_types  = [
    "AWS::CloudWatch::Metric",
    "AWS::Logs::LogGroup",
    "AWS::XRay::Trace"
  ]
  sink_identifier = var.monitoring_sink_arn  # ARN from the monitoring account

  tags = {
    Name = "link-to-monitoring-account"
  }
}
```

## Step 3: Cross-Account CloudWatch Alarm (Legacy Method)

```hcl
# Alternative: Cross-account metric sharing via sharing policies
# Configure a sharing policy in the source account
resource "aws_cloudwatch_metric_stream" "to_monitoring" {
  name          = "${var.project_name}-metric-stream"
  role_arn      = aws_iam_role.metric_stream.arn
  firehose_arn  = var.monitoring_firehose_arn
  output_format = "json"

  # Stream specific namespaces
  include_filter {
    namespace = "AWS/Lambda"
  }

  include_filter {
    namespace = "AWS/RDS"
  }

  include_filter {
    namespace = "AWS/ApiGateway"
  }

  tags = {
    Name = "${var.project_name}-metric-stream"
  }
}

resource "aws_iam_role" "metric_stream" {
  name = "${var.project_name}-metric-stream-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action    = "sts:AssumeRole"
      Effect    = "Allow"
      Principal = { Service = "streams.metrics.cloudwatch.amazonaws.com" }
    }]
  })
}

resource "aws_iam_role_policy" "metric_stream" {
  name = "firehose-put"
  role = aws_iam_role.metric_stream.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["firehose:PutRecord", "firehose:PutRecordBatch"]
      Resource = var.monitoring_firehose_arn
    }]
  })
}
```

## Step 4: Deploy

```bash
# Deploy monitoring account resources
cd monitoring-account
tofu init
tofu apply
# Note the sink_arn output

# Deploy source account resources with the sink ARN
cd ../source-account
tofu init
tofu apply -var="monitoring_sink_arn=<sink-arn>"
```

## Conclusion

CloudWatch cross-account observability using OAM (Observability Access Manager) is the recommended approach for multi-account AWS Organizations. Once linked, the monitoring account can query metrics, search log groups, and view traces from all source accounts in a single console view. Link accounts by function (production, staging, development) or team to scope monitoring access appropriately.
