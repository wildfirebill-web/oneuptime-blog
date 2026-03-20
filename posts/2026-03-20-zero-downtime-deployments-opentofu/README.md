# How to Implement Zero-Downtime Deployments with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Zero Downtime, Rolling Deployment, ECS, Kubernetes, Infrastructure as Code, Reliability

Description: Learn how to configure zero-downtime deployments using OpenTofu by implementing rolling updates, connection draining, and health check grace periods across ECS and Kubernetes.

---

Zero-downtime deployments require that new instances are healthy and serving traffic before old instances are terminated. OpenTofu configuration controls the deployment strategy - but the defaults are rarely sufficient for production. This guide shows the specific settings that make the difference.

## ECS Rolling Deployment with Zero Downtime

```hcl
# main.tf

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# ALB Target Group with connection draining
resource "aws_lb_target_group" "app" {
  name        = "${var.service_name}-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"  # Required for Fargate

  # Health check settings
  health_check {
    path                = "/health"
    healthy_threshold   = 2    # 2 successful checks = healthy
    unhealthy_threshold = 2    # 2 failed checks = unhealthy
    interval            = 10   # Check every 10 seconds
    timeout             = 5    # Consider timeout a failure after 5 seconds
    matcher             = "200"
  }

  # Connection draining - allow in-flight requests to complete
  deregistration_delay = 60  # Wait 60 seconds before terminating deregistered targets
}

# ECS Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = var.service_name
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.execution.arn
  task_role_arn            = aws_iam_role.task.arn

  container_definitions = jsonencode([{
    name  = var.service_name
    image = var.app_image
    portMappings = [{
      containerPort = 8080
    }]

    # Stop the container gracefully before force-killing
    stopTimeout = 60  # Seconds to wait for graceful shutdown

    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
      interval    = 10
      timeout     = 5
      retries     = 3
      startPeriod = 30  # Don't check for first 30 seconds (startup grace period)
    }
  }])
}

# ECS Service with rolling deployment configuration
resource "aws_ecs_service" "app" {
  name            = var.service_name
  cluster         = var.ecs_cluster_arn
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  # Rolling deployment settings
  deployment_maximum_percent         = 200  # Allow up to 2x instances during deployment
  deployment_minimum_healthy_percent = 100  # Never go below 100% capacity

  # Health check grace period - prevents new tasks from being killed during startup
  health_check_grace_period_seconds = 120

  # Circuit breaker prevents stuck deployments
  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = var.service_name
    container_port   = 8080
  }

  network_configuration {
    subnets         = var.private_subnet_ids
    security_groups = [aws_security_group.app.id]
  }
}
```

## Kubernetes Rolling Update Configuration

```hcl
# kubernetes_zero_downtime.tf
resource "kubernetes_deployment" "app" {
  metadata {
    name      = var.service_name
    namespace = var.namespace
  }

  spec {
    replicas = var.desired_count

    # Rolling update strategy
    strategy {
      type = "RollingUpdate"

      rolling_update {
        # Maximum extra pods during rollout (25% of desired_count)
        max_surge = "25%"

        # Minimum available pods during rollout (must stay at 100% capacity)
        max_unavailable = "0"
      }
    }

    selector {
      match_labels = {
        app = var.service_name
      }
    }

    template {
      metadata {
        labels = {
          app = var.service_name
        }
      }

      spec {
        # Graceful shutdown - wait for in-flight requests to complete
        termination_grace_period_seconds = 60

        container {
          name  = var.service_name
          image = var.app_image

          # Liveness probe - restart unhealthy containers
          liveness_probe {
            http_get {
              path = "/health"
              port = 8080
            }
            initial_delay_seconds = 30
            period_seconds        = 10
            failure_threshold     = 3
          }

          # Readiness probe - only route traffic when ready
          readiness_probe {
            http_get {
              path = "/ready"
              port = 8080
            }
            initial_delay_seconds = 10
            period_seconds        = 5
            failure_threshold     = 3
          }

          lifecycle {
            # Pre-stop hook - ensures in-flight requests complete before termination
            pre_stop {
              exec {
                command = ["/bin/sh", "-c", "sleep 15"]
              }
            }
          }
        }
      }
    }
  }
}

# Pod Disruption Budget - prevents simultaneous pod terminations
resource "kubernetes_pod_disruption_budget_v1" "app" {
  metadata {
    name      = "${var.service_name}-pdb"
    namespace = var.namespace
  }

  spec {
    min_available = "50%"  # At least 50% of pods must remain available

    selector {
      match_labels = {
        app = var.service_name
      }
    }
  }
}
```

## Best Practices

- Set `max_unavailable = 0` or `deployment_minimum_healthy_percent = 100` - never allow capacity to drop below 100% during deployments.
- Configure `deregistration_delay` on ALB target groups to allow in-flight requests to complete before terminating instances.
- Use container `startPeriod` in health checks to give applications time to initialize before health checks begin.
- Implement application-level graceful shutdown handlers that complete in-flight requests before exiting.
- Use Pod Disruption Budgets in Kubernetes to prevent cluster maintenance from taking down too many instances at once.
