# How to Configure ECS Container Insights with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, ECS, Container Insights, CloudWatch, Monitoring, Infrastructure as Code

Description: Learn how to configure ECS Container Insights with OpenTofu to collect CPU, memory, network, and storage metrics for ECS clusters, services, and tasks in CloudWatch.

## Introduction

ECS Container Insights uses CloudWatch to collect granular metrics for ECS clusters, services, and individual tasks including CPU utilization, memory utilization, network I/O, and storage I/O. Unlike basic CloudWatch metrics, Container Insights provides task-level granularity and enables alerting on individual service or task performance.

## Prerequisites

- OpenTofu v1.6+
- An ECS cluster
- AWS credentials with ECS and CloudWatch permissions

## Step 1: Enable Container Insights on ECS Cluster

```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.project_name}-cluster"

  # Enable Container Insights
  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Name = "${var.project_name}-cluster"
  }
}

# Account-level default for Container Insights
resource "aws_ecs_account_setting_default" "container_insights" {
  name  = "containerInsights"
  value = "enabled"  # Enable for all new clusters in this account
}
```

## Step 2: CloudWatch Alarms Using Container Insights Metrics

```hcl
# Alert when service CPU exceeds 80%
resource "aws_cloudwatch_metric_alarm" "ecs_cpu_high" {
  alarm_name          = "${var.project_name}-ecs-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CpuUtilized"
  namespace           = "ECS/ContainerInsights"
  period              = 300
  statistic           = "Average"
  threshold           = 80.0

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = "${var.project_name}-app"
  }

  alarm_description = "ECS service CPU utilization above 80%"
  alarm_actions     = [var.sns_topic_arn]
  ok_actions        = [var.sns_topic_arn]

  treat_missing_data = "notBreaching"
}

# Alert when service memory exceeds 85%
resource "aws_cloudwatch_metric_alarm" "ecs_memory_high" {
  alarm_name          = "${var.project_name}-ecs-memory-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "MemoryUtilized"
  namespace           = "ECS/ContainerInsights"
  period              = 300
  statistic           = "Average"
  threshold           = var.memory_threshold_mb  # Absolute MB threshold

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = "${var.project_name}-app"
  }

  alarm_actions = [var.sns_topic_arn]
}

# Alert when running task count drops below desired
resource "aws_cloudwatch_metric_alarm" "ecs_task_count_low" {
  alarm_name          = "${var.project_name}-ecs-tasks-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = 2
  metric_name         = "RunningTaskCount"
  namespace           = "ECS/ContainerInsights"
  period              = 60
  statistic           = "Average"
  threshold           = var.min_task_count

  dimensions = {
    ClusterName = aws_ecs_cluster.main.name
    ServiceName = "${var.project_name}-app"
  }

  alarm_description = "ECS running task count dropped below minimum"
  alarm_actions     = [var.critical_sns_topic_arn]
  treat_missing_data = "breaching"
}
```

## Step 3: Container Insights Dashboard

```hcl
resource "aws_cloudwatch_dashboard" "ecs_insights" {
  dashboard_name = "${var.project_name}-ecs-insights"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "ECS Service CPU & Memory"
          view   = "timeSeries"
          period = 300
          metrics = [
            ["ECS/ContainerInsights", "CpuUtilized",
             "ClusterName", aws_ecs_cluster.main.name,
             "ServiceName", "${var.project_name}-app",
             { stat = "Average", yAxis = "left", label = "CPU (cores)" }],
            ["ECS/ContainerInsights", "MemoryUtilized",
             "ClusterName", aws_ecs_cluster.main.name,
             "ServiceName", "${var.project_name}-app",
             { stat = "Average", yAxis = "right", label = "Memory (MB)" }]
          ]
        }
      },
      {
        type   = "metric"
        x      = 12
        y      = 0
        width  = 12
        height = 6
        properties = {
          title  = "Task Count"
          view   = "timeSeries"
          period = 60
          stat   = "Average"
          metrics = [
            ["ECS/ContainerInsights", "RunningTaskCount",
             "ClusterName", aws_ecs_cluster.main.name,
             "ServiceName", "${var.project_name}-app"]
          ]
        }
      },
      {
        type   = "metric"
        x      = 0
        y      = 6
        width  = 24
        height = 6
        properties = {
          title  = "Network I/O"
          view   = "timeSeries"
          period = 60
          metrics = [
            ["ECS/ContainerInsights", "NetworkRxBytes",
             "ClusterName", aws_ecs_cluster.main.name,
             "ServiceName", "${var.project_name}-app",
             { stat = "Sum", label = "Network In" }],
            ["ECS/ContainerInsights", "NetworkTxBytes",
             "ClusterName", aws_ecs_cluster.main.name,
             "ServiceName", "${var.project_name}-app",
             { stat = "Sum", label = "Network Out" }]
          ]
        }
      }
    ]
  })
}
```

## Step 4: Deploy

```bash
tofu init
tofu plan
tofu apply

# View Container Insights metrics
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=my-project-cluster Name=ServiceName,Value=my-project-app \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

## Conclusion

Container Insights is essential for ECS Fargate workloads where you can't access underlying EC2 instances—it's the only way to get task-level CPU and memory metrics. Note that Container Insights charges for custom CloudWatch metrics ($0.30/metric/month); monitor the metric count if cost is a concern. Alert on `RunningTaskCount` dropping below expected values—this catches service degradation that pure CPU/memory metrics miss.
