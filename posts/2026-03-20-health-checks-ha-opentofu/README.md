# How to Configure Health Checks for High Availability with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: High Availability, Health Checks, OpenTofu, ALB, Route53, GCP, Azure

Description: Learn how to configure comprehensive health checks across AWS, Azure, and GCP using OpenTofu for effective load balancer routing and automatic instance replacement.

## Overview

Health checks are the foundation of high availability - they detect failures and trigger routing changes or instance replacement. OpenTofu configures layered health checks: shallow for load balancer routing, deep for application readiness, and synthetic for external monitoring.

## Step 1: AWS ALB Health Checks

```hcl
# main.tf - Multi-level health checks

# Shallow health check for load balancer (fast, lightweight)
resource "aws_lb_target_group" "shallow" {
  name     = "app-tg-shallow"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  health_check {
    path                = "/health/live"   # Just checks if process is running
    healthy_threshold   = 2
    unhealthy_threshold = 2               # Fast deregistration
    timeout             = 3
    interval            = 15
    matcher             = "200"
  }
}

# Deep health check for dependency validation
resource "aws_lb_target_group" "deep" {
  name     = "app-tg-deep"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  health_check {
    path                = "/health/ready"  # Checks DB, cache, dependencies
    healthy_threshold   = 3
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    matcher             = "200"
  }
}
```

## Step 2: Route53 Health Checks with CloudWatch

```hcl
# Calculated health check combining multiple checks
resource "aws_route53_health_check" "composite" {
  type                    = "CALCULATED"
  child_health_threshold  = 2  # Need at least 2 of 3 regions healthy
  child_healthchecks      = [
    aws_route53_health_check.region_us.id,
    aws_route53_health_check.region_eu.id,
    aws_route53_health_check.region_ap.id
  ]
}

# Metric-based health check using CloudWatch alarm
resource "aws_route53_health_check" "cloudwatch_alarm" {
  type                            = "CLOUDWATCH_METRIC"
  cloudwatch_alarm_name           = aws_cloudwatch_metric_alarm.app_5xx_rate.alarm_name
  cloudwatch_alarm_region         = "us-east-1"
  insufficient_data_health_status = "Unhealthy"
}

# 5xx rate alarm
resource "aws_cloudwatch_metric_alarm" "app_5xx_rate" {
  alarm_name          = "app-5xx-rate-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  threshold           = 5  # > 5% error rate

  metric_query {
    id          = "error_rate"
    expression  = "(errors / total) * 100"
    label       = "5xx Error Rate"
    return_data = true
  }

  metric_query {
    id = "errors"
    metric {
      metric_name = "HTTPCode_Target_5XX_Count"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = aws_lb.app.arn_suffix }
    }
  }

  metric_query {
    id = "total"
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 60
      stat        = "Sum"
      dimensions  = { LoadBalancer = aws_lb.app.arn_suffix }
    }
  }
}
```

## Step 3: GCP Health Check Configuration

```hcl
# GCP TCP health check (fastest)
resource "google_compute_health_check" "tcp" {
  name               = "tcp-health-check"
  check_interval_sec = 5
  timeout_sec        = 3
  healthy_threshold  = 2
  unhealthy_threshold = 3

  tcp_health_check {
    port = 8080
  }
}

# GCP HTTP health check with custom headers
resource "google_compute_health_check" "http_deep" {
  name               = "http-deep-health-check"
  check_interval_sec = 15
  timeout_sec        = 5
  healthy_threshold  = 2
  unhealthy_threshold = 3

  http_health_check {
    port         = 8080
    request_path = "/health/ready"
    request_headers = "X-Health-Check: gcp-lb"
    response     = "OK"  # Expected response body
  }
}
```

## Step 4: Kubernetes Probes

```hcl
# Kubernetes deployment with three probe types
resource "kubernetes_deployment" "app" {
  metadata {
    name      = "app"
    namespace = "production"
  }

  spec {
    template {
      spec {
        container {
          name  = "app"
          image = "app:latest"

          # Startup probe - gives slow containers time to start
          startup_probe {
            http_get {
              path = "/health/start"
              port = 8080
            }
            failure_threshold = 30  # 30 * 10s = 5 minutes max startup
            period_seconds    = 10
          }

          # Liveness probe - restart if unhealthy
          liveness_probe {
            http_get {
              path = "/health/live"
              port = 8080
            }
            initial_delay_seconds = 0  # Startup probe handles initial wait
            period_seconds        = 15
            failure_threshold     = 3
          }

          # Readiness probe - remove from load balancer if not ready
          readiness_probe {
            http_get {
              path = "/health/ready"
              port = 8080
            }
            period_seconds    = 10
            failure_threshold = 3
            success_threshold = 1
          }
        }
      }
    }
  }
}
```

## Summary

Health check configuration with OpenTofu uses layered checks for different purposes: liveness checks trigger container restarts, readiness checks control load balancer routing, and startup checks give slow-starting applications extra initialization time. AWS Route53 calculated health checks aggregate multiple regional checks, requiring a quorum of healthy regions before failover. GCP health checks with expected response validation prevent false positives where an application returns 200 but serves error content.
