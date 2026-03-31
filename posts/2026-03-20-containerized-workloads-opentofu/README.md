# How to Deploy Containerized Workloads with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Docker, ECS, Kubernetes, Container, Infrastructure as Code

Description: Learn how to provision containerized workloads on AWS ECS and Kubernetes using OpenTofu for repeatable container deployments.

---

OpenTofu can provision the entire container infrastructure stack - ECS clusters, task definitions, Kubernetes deployments, and the supporting networking - enabling fully automated container deployments.

---

## Deploy to AWS ECS with Fargate

```hcl
# ECS Cluster

resource "aws_ecs_cluster" "main" {
  name = "my-cluster"
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "my-app"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  container_definitions = jsonencode([{
    name  = "app"
    image = "nginx:alpine"
    portMappings = [{
      containerPort = 80
      protocol      = "tcp"
    }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"  = aws_cloudwatch_log_group.app.name
        "awslogs-region" = "us-east-1"
        "awslogs-stream-prefix" = "app"
      }
    }
  }])
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "my-app"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 2
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = aws_subnet.private[*].id
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 80
  }
}
```

---

## Deploy to Kubernetes with the Kubernetes Provider

```hcl
provider "kubernetes" {
  config_path = "~/.kube/config"
}

resource "kubernetes_deployment" "app" {
  metadata {
    name      = "my-app"
    namespace = "default"
  }

  spec {
    replicas = 3

    selector {
      match_labels = {
        app = "my-app"
      }
    }

    template {
      metadata {
        labels = {
          app = "my-app"
        }
      }

      spec {
        container {
          name  = "app"
          image = "nginx:alpine"

          port {
            container_port = 80
          }

          resources {
            limits = {
              cpu    = "250m"
              memory = "128Mi"
            }
          }
        }
      }
    }
  }
}
```

---

## Summary

Use `aws_ecs_task_definition` and `aws_ecs_service` to deploy containers on AWS Fargate, or the Kubernetes provider's `kubernetes_deployment` for Kubernetes clusters. Both approaches let you version-control your container configurations, set resource limits, configure networking, and integrate with load balancers - all through OpenTofu.
