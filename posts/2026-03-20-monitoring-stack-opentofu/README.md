# How to Build a Monitoring Stack with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Monitoring, CloudWatch, Prometheus, Grafana, Infrastructure as Code

Description: Learn how to build a monitoring stack on AWS with OpenTofu, including CloudWatch dashboards, alarms, Managed Prometheus, and Managed Grafana.

## Introduction

A monitoring stack needs metrics collection, alerting, and visualization. On AWS, you can use CloudWatch for basic metrics and alarms, Amazon Managed Service for Prometheus (AMP) for Kubernetes metrics, and Amazon Managed Grafana for visualization. This guide provisions all three with OpenTofu.

## CloudWatch Log Groups and Metric Filters

```hcl
resource "aws_cloudwatch_log_group" "application" {
  name              = "/myapp/${var.environment}/application"
  retention_in_days = var.environment == "prod" ? 90 : 14

  tags = {
    Application = "myapp"
    Environment = var.environment
  }
}

# Extract error count from application logs

resource "aws_cloudwatch_log_metric_filter" "error_count" {
  name           = "myapp-error-count"
  pattern        = "[timestamp, level=\"ERROR\", ...]"
  log_group_name = aws_cloudwatch_log_group.application.name

  metric_transformation {
    name      = "ErrorCount"
    namespace = "MyApp/${var.environment}"
    value     = "1"
    unit      = "Count"
  }
}

resource "aws_cloudwatch_metric_alarm" "high_errors" {
  alarm_name          = "myapp-high-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ErrorCount"
  namespace           = "MyApp/${var.environment}"
  period              = 60
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Application error rate exceeded 10/minute"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  ok_actions          = [aws_sns_topic.alerts.arn]
  treat_missing_data  = "notBreaching"
}
```

## CloudWatch Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "application" {
  dashboard_name = "myapp-${var.environment}"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        width  = 12
        height = 6
        properties = {
          title  = "API Request Rate"
          period = 60
          stat   = "Sum"
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", aws_lb.app.arn_suffix]
          ]
        }
      },
      {
        type   = "metric"
        width  = 12
        height = 6
        properties = {
          title  = "API Error Rate"
          period = 60
          stat   = "Sum"
          metrics = [
            ["AWS/ApplicationELB", "HTTPCode_Target_5XX_Count", "LoadBalancer", aws_lb.app.arn_suffix]
          ]
        }
      },
      {
        type   = "alarm"
        width  = 24
        height = 4
        properties = {
          title  = "Alarms"
          alarms = [aws_cloudwatch_metric_alarm.high_errors.arn]
        }
      }
    ]
  })
}
```

## Amazon Managed Prometheus

```hcl
resource "aws_prometheus_workspace" "main" {
  alias = "myapp-${var.environment}"

  logging_configuration {
    log_group_arn = "${aws_cloudwatch_log_group.prometheus.arn}:*"
  }
}

resource "aws_cloudwatch_log_group" "prometheus" {
  name              = "/aws/prometheus/myapp-${var.environment}"
  retention_in_days = 30
}

# IAM role for EKS pods to remote-write to AMP
resource "aws_iam_role" "prometheus_write" {
  name = "prometheus-remote-write"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = "sts:AssumeRoleWithWebIdentity"
      Principal = {
        Federated = module.eks.oidc_provider_arn
      }
      Condition = {
        StringEquals = {
          "${module.eks.oidc_provider}:sub" = "system:serviceaccount:monitoring:prometheus"
        }
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "prometheus_write" {
  role       = aws_iam_role.prometheus_write.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonPrometheusRemoteWriteAccess"
}
```

## Amazon Managed Grafana

```hcl
resource "aws_grafana_workspace" "main" {
  name                     = "myapp-${var.environment}-grafana"
  account_access_type      = "CURRENT_ACCOUNT"
  authentication_providers = ["AWS_SSO"]
  permission_type          = "SERVICE_MANAGED"
  role_arn                 = aws_iam_role.grafana.arn

  data_sources = ["AMAZON_OPENSEARCH_SERVICE", "CLOUDWATCH", "PROMETHEUS"]
}

resource "aws_grafana_workspace_api_key" "main" {
  key_name        = "terraform-provisioning"
  key_role        = "ADMIN"
  seconds_to_live = 3600
  workspace_id    = aws_grafana_workspace.main.id
}
```

## SNS Alerting Topic

```hcl
resource "aws_sns_topic" "alerts" {
  name = "myapp-${var.environment}-alerts"
}

# Email subscription
resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# PagerDuty subscription via HTTPS
resource "aws_sns_topic_subscription" "pagerduty" {
  count     = var.environment == "prod" ? 1 : 0
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "https"
  endpoint  = var.pagerduty_sns_endpoint
}
```

## Summary

A complete monitoring stack combines CloudWatch for AWS-native metrics and log-based metrics, Amazon Managed Prometheus for Kubernetes/container metrics, Amazon Managed Grafana for visualization, and SNS for alert routing. Use CloudWatch Log Metric Filters to extract business metrics (error rates, login counts, payment amounts) from application logs, and connect alarms to PagerDuty for production on-call workflows.
