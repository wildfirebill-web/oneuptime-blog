# How to Create ECS Services with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, ECS, Terraform, IaC, DevOps, Container

Description: Learn how to create AWS ECS clusters, task definitions, and services with OpenTofu for running containerized applications on Fargate.

## Introduction

ECS (Elastic Container Service) with Fargate runs containers without managing EC2 instances. An ECS deployment in OpenTofu involves a cluster, task definition (describing the container), ECS service (managing running instances), security group, and IAM roles. This guide covers deploying a containerized application on ECS Fargate.

## ECS Cluster

```hcl
resource "aws_ecs_cluster" "main" {
  name = "${var.environment}-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = { Name = "${var.environment}-cluster" }
}
```

## IAM Roles

```hcl
# Task execution role (used by ECS to pull images and write logs)

resource "aws_iam_role" "task_execution" {
  name = "${var.environment}-ecs-task-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "task_execution" {
  role       = aws_iam_role.task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Task role (used by the application code inside the container)
resource "aws_iam_role" "task" {
  name = "${var.environment}-ecs-task-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## CloudWatch Log Group

```hcl
resource "aws_cloudwatch_log_group" "app" {
  name              = "/ecs/${var.environment}/${var.service_name}"
  retention_in_days = 30
}
```

## Task Definition

```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "${var.environment}-${var.service_name}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = var.task_cpu
  memory                   = var.task_memory
  execution_role_arn       = aws_iam_role.task_execution.arn
  task_role_arn            = aws_iam_role.task.arn

  container_definitions = jsonencode([{
    name  = var.service_name
    image = "${var.ecr_repository_url}:${var.image_tag}"

    portMappings = [{
      containerPort = var.container_port
      protocol      = "tcp"
    }]

    environment = [
      { name = "ENVIRONMENT", value = var.environment },
      { name = "PORT",        value = tostring(var.container_port) }
    ]

    secrets = [
      {
        name      = "DATABASE_URL"
        valueFrom = aws_secretsmanager_secret.db_url.arn
      }
    ]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        awslogs-group         = aws_cloudwatch_log_group.app.name
        awslogs-region        = var.region
        awslogs-stream-prefix = "ecs"
      }
    }

    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:${var.container_port}/health || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }
  }])

  tags = { Name = "${var.environment}-${var.service_name}" }
}
```

## Security Group for ECS Tasks

```hcl
resource "aws_security_group" "ecs_tasks" {
  name   = "${var.environment}-${var.service_name}-tasks-sg"
  vpc_id = var.vpc_id

  ingress {
    description     = "From ALB"
    from_port       = var.container_port
    to_port         = var.container_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## ECS Service

```hcl
resource "aws_ecs_service" "app" {
  name            = "${var.environment}-${var.service_name}"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = var.service_name
    container_port   = var.container_port
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  lifecycle {
    ignore_changes = [task_definition, desired_count]
    # Ignore task_definition to allow external deployment tools to update
  }

  depends_on = [aws_lb_listener.https]
}
```

## Outputs

```hcl
output "cluster_arn"     { value = aws_ecs_cluster.main.arn }
output "cluster_name"    { value = aws_ecs_cluster.main.name }
output "service_name"    { value = aws_ecs_service.app.name }
output "task_definition" { value = aws_ecs_task_definition.app.arn }
```

## Conclusion

A production ECS Fargate service requires a cluster, IAM execution and task roles, CloudWatch log group, task definition, security group, and the service itself. Use `secrets` in the container definition (not `environment`) for sensitive values like database URLs. Enable the deployment circuit breaker to automatically roll back failed deployments. Ignore `task_definition` changes in the service lifecycle if you use a separate CI/CD pipeline for image updates.
