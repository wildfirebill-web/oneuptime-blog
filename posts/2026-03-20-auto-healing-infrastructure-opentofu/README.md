# How to Set Up Auto-Healing Infrastructure with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Auto-Healing, High Availability, OpenTofu, Auto Scaling, Kubernetes, Self-Healing

Description: Learn how to configure auto-healing infrastructure using OpenTofu that automatically detects and replaces failed instances, pods, and services without manual intervention.

## Overview

Auto-healing infrastructure automatically detects unhealthy components and replaces them. OpenTofu configures auto-healing at multiple levels: EC2 instance replacement via ASG, Kubernetes pod restarts, and managed database failover.

## Step 1: Auto Scaling Group with Instance Recovery

```hcl
# main.tf - ASG with comprehensive auto-healing

resource "aws_autoscaling_group" "self_healing" {
  name                = "self-healing-asg"
  vpc_zone_identifier = module.vpc.private_subnets
  target_group_arns   = [aws_lb_target_group.app.arn]

  min_size         = 3
  max_size         = 30
  desired_capacity = 6

  # Use ELB health checks (not just EC2 status)
  health_check_type         = "ELB"
  health_check_grace_period = 300

  # Enable instance refresh circuit breaker
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 90
      auto_rollback          = true  # Rollback if health check fails
    }
    triggers = ["tag"]
  }

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Version"
    value               = var.app_version
    propagate_at_launch = true
  }
}

# CloudWatch alarm to trigger additional healing actions
resource "aws_cloudwatch_metric_alarm" "unhealthy_hosts" {
  alarm_name          = "asg-unhealthy-hosts"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "UnHealthyHostCount"
  namespace           = "AWS/ApplicationELB"
  period              = 60
  statistic           = "Average"
  threshold           = 1

  dimensions = {
    TargetGroup  = aws_lb_target_group.app.arn_suffix
    LoadBalancer = aws_lb.app.arn_suffix
  }

  alarm_actions = [aws_sns_topic.alerts.arn]
}
```

## Step 2: EC2 Instance Recovery

```hcl
# CloudWatch alarm triggers EC2 auto-recovery for system failures
resource "aws_cloudwatch_metric_alarm" "instance_recovery" {
  alarm_name          = "ec2-instance-recovery"
  comparison_operator = "GreaterThanOrEqualToThreshold"
  evaluation_periods  = 2
  metric_name         = "StatusCheckFailed_System"
  namespace           = "AWS/EC2"
  period              = 60
  statistic           = "Maximum"
  threshold           = 1

  dimensions = {
    InstanceId = aws_instance.app.id
  }

  # EC2 native recovery action - migrates to healthy hardware
  alarm_actions = ["arn:aws:automate:us-east-1:ec2:recover"]

  # Also send notification
  alarm_actions = [
    "arn:aws:automate:us-east-1:ec2:recover",
    aws_sns_topic.alerts.arn
  ]
}
```

## Step 3: Kubernetes Self-Healing Configuration

```hcl
# Kubernetes Deployment with liveness probes for auto-restart
resource "kubernetes_deployment" "self_healing" {
  metadata {
    name      = "self-healing-app"
    namespace = "production"
  }

  spec {
    replicas = 3

    strategy {
      type = "RollingUpdate"
      rolling_update {
        max_unavailable = "0"
        max_surge       = "1"
      }
    }

    template {
      spec {
        # Restart policy (default is Always for Deployments)
        restart_policy = "Always"

        container {
          name  = "app"
          image = "app:latest"

          liveness_probe {
            http_get {
              path = "/health/live"
              port = 8080
            }
            initial_delay_seconds = 30
            period_seconds        = 10
            failure_threshold     = 3
          }

          readiness_probe {
            http_get {
              path = "/health/ready"
              port = 8080
            }
            period_seconds    = 5
            failure_threshold = 3
          }

          resources {
            requests = {
              cpu    = "500m"
              memory = "512Mi"
            }
            limits = {
              cpu    = "2000m"
              memory = "1Gi"
            }
          }
        }
      }
    }
  }
}

# Horizontal Pod Autoscaler ensures desired replicas maintained
resource "kubernetes_horizontal_pod_autoscaler_v2" "self_healing" {
  metadata {
    name      = "self-healing-app-hpa"
    namespace = "production"
  }

  spec {
    scale_target_ref {
      api_version = "apps/v1"
      kind        = "Deployment"
      name        = kubernetes_deployment.self_healing.metadata[0].name
    }

    min_replicas = 3
    max_replicas = 30

    metric {
      type = "Resource"
      resource {
        name = "cpu"
        target {
          type                = "Utilization"
          average_utilization = 70
        }
      }
    }
  }
}
```

## Step 4: RDS Auto-Healing with Multi-AZ

```hcl
# RDS Multi-AZ for automatic database failover
resource "aws_db_instance" "auto_healing" {
  identifier     = "app-db-auto-healing"
  engine         = "postgres"
  instance_class = "db.r6g.large"
  multi_az       = true

  # Enable enhanced monitoring for faster failure detection
  monitoring_interval = 15
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn

  # Performance Insights for diagnostic data
  performance_insights_enabled = true
}

# CloudWatch alarm for RDS failover events
resource "aws_cloudwatch_event_rule" "rds_failover" {
  name        = "rds-failover-event"
  description = "Capture RDS failover events"
  event_pattern = jsonencode({
    source      = ["aws.rds"]
    detail-type = ["RDS DB Instance Event"]
    detail = {
      EventID = ["RDS-EVENT-0049"]  # Multi-AZ failover complete
    }
  })
}
```

## Summary

Auto-healing infrastructure configured with OpenTofu operates across multiple layers: ASG replaces terminated EC2 instances within minutes, CloudWatch EC2 recovery migrates failed instances to healthy hardware without data loss, and Kubernetes liveness probes restart containers that become deadlocked. RDS Multi-AZ provides 60-120 second automatic database failover. Together, these mechanisms ensure most failure scenarios are resolved automatically without paging on-call engineers.
