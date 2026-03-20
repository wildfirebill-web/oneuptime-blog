# How to Create a CloudWatch Alarm with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, CloudWatch, Monitoring, Infrastructure as Code

Description: Learn how to create AWS CloudWatch alarms with OpenTofu to monitor metrics, trigger SNS notifications, and automate responses to infrastructure events.

## Introduction

CloudWatch alarms watch a single metric and take actions when it crosses a threshold. OpenTofu makes it straightforward to define alarms as code alongside the resources they monitor, ensuring monitoring is never an afterthought.

## SNS Topic for Notifications

```hcl
resource "aws_sns_topic" "alarms" {
  name = "${var.name}-alarms"
}

resource "aws_sns_topic_subscription" "email" {
  topic_arn = aws_sns_topic.alarms.arn
  protocol  = "email"
  endpoint  = var.alert_email
}
```

## Basic Metric Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.name}-cpu-high"
  alarm_description   = "EC2 CPU utilisation is above ${var.cpu_threshold}%"

  # Metric to monitor
  namespace   = "AWS/EC2"
  metric_name = "CPUUtilization"
  dimensions = {
    InstanceId = aws_instance.web.id
  }

  # Alarm conditions
  comparison_operator = "GreaterThanThreshold"
  threshold           = var.cpu_threshold  # e.g., 80
  evaluation_periods  = 2
  period              = 300  # seconds (5 minutes)
  statistic           = "Average"

  # Alarm state transitions
  alarm_actions             = [aws_sns_topic.alarms.arn]
  ok_actions                = [aws_sns_topic.alarms.arn]
  insufficient_data_actions = []

  treat_missing_data = "notBreaching"

  tags = { Name = "${var.name}-cpu-high" }
}
```

## RDS Alarm

```hcl
resource "aws_cloudwatch_metric_alarm" "rds_connections_high" {
  alarm_name        = "${var.name}-rds-connections"
  alarm_description = "RDS connection count is critically high"
  namespace         = "AWS/RDS"
  metric_name       = "DatabaseConnections"

  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.identifier
  }

  comparison_operator = "GreaterThanThreshold"
  threshold           = 80
  evaluation_periods  = 3
  period              = 60
  statistic           = "Average"

  alarm_actions = [aws_sns_topic.alarms.arn]
  ok_actions    = [aws_sns_topic.alarms.arn]
}
```

## Composite Alarm (Multiple Conditions)

```hcl
# Individual alarms
resource "aws_cloudwatch_metric_alarm" "cpu" {
  alarm_name          = "${var.name}-cpu"
  namespace           = "AWS/EC2"
  metric_name         = "CPUUtilization"
  dimensions          = { InstanceId = aws_instance.web.id }
  comparison_operator = "GreaterThanThreshold"
  threshold           = 90
  evaluation_periods  = 1
  period              = 60
  statistic           = "Average"
}

resource "aws_cloudwatch_metric_alarm" "memory" {
  alarm_name          = "${var.name}-memory"
  namespace           = "CWAgent"
  metric_name         = "mem_used_percent"
  dimensions          = { InstanceId = aws_instance.web.id }
  comparison_operator = "GreaterThanThreshold"
  threshold           = 90
  evaluation_periods  = 1
  period              = 60
  statistic           = "Average"
}

# Composite alarm fires when BOTH conditions are true
resource "aws_cloudwatch_composite_alarm" "critical" {
  alarm_name        = "${var.name}-critical"
  alarm_description = "CPU AND memory are both critically high"

  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.cpu.alarm_name}) AND ALARM(${aws_cloudwatch_metric_alarm.memory.alarm_name})"

  alarm_actions = [aws_sns_topic.alarms.arn]
}
```

## Auto Scaling Trigger

```hcl
# Trigger scale-out when CPU stays above 70% for 2 evaluation periods
resource "aws_cloudwatch_metric_alarm" "scale_out" {
  alarm_name          = "${var.name}-scale-out"
  namespace           = "AWS/ECS"
  metric_name         = "CPUUtilization"
  dimensions          = { ServiceName = var.ecs_service_name, ClusterName = var.cluster_name }
  comparison_operator = "GreaterThanThreshold"
  threshold           = 70
  evaluation_periods  = 2
  period              = 60
  statistic           = "Average"

  alarm_actions = [aws_appautoscaling_policy.scale_out.arn]
}
```

## Conclusion

CloudWatch alarms defined in OpenTofu are always co-located with the resources they monitor. Use composite alarms to reduce alert noise, and always configure both `alarm_actions` and `ok_actions` so your team knows when an issue resolves automatically.
