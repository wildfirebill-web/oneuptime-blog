# How to Deploy a Java Spring Boot Application with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Java, Spring Boot, ECS, AWS, Infrastructure as Code

Description: Learn how to deploy a Java Spring Boot application on AWS using OpenTofu, with ECS Fargate for containers, RDS for the database, and proper JVM configuration.

## Introduction

Spring Boot applications run well on ECS Fargate with proper JVM memory configuration. This guide deploys a Spring Boot application with ECS Fargate, RDS PostgreSQL, Application Load Balancer, and Spring Boot Actuator health checks.

## ECR Repository

```hcl
resource "aws_ecr_repository" "spring_boot" {
  name                 = "myapp-spring-boot"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

## ECS Task Definition with JVM Tuning

Spring Boot needs adequate memory for the JVM. Set `-Xmx` to 75% of the container memory.

```hcl
resource "aws_ecs_task_definition" "spring_boot" {
  family                   = "myapp-spring-boot"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "1024"   # 1 vCPU
  memory                   = "2048"   # 2 GB
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([{
    name  = "spring-boot-app"
    image = "${aws_ecr_repository.spring_boot.repository_url}:${var.app_version}"

    # JVM memory settings: use ~75% of container memory for heap
    command = [
      "java",
      "-Xms256m",
      "-Xmx1536m",
      "-XX:+UseContainerSupport",
      "-XX:MaxRAMPercentage=75.0",
      "-Dspring.profiles.active=${var.environment}",
      "-jar",
      "/app/app.jar"
    ]

    environment = [
      { name = "SPRING_PROFILES_ACTIVE", value = var.environment },
      { name = "SERVER_PORT",            value = "8080" },
    ]

    secrets = [
      { name = "SPRING_DATASOURCE_URL",      valueFrom = aws_secretsmanager_secret.db_url.arn },
      { name = "SPRING_DATASOURCE_USERNAME", valueFrom = aws_secretsmanager_secret.db_user.arn },
      { name = "SPRING_DATASOURCE_PASSWORD", valueFrom = aws_secretsmanager_secret.db_pass.arn },
    ]

    portMappings = [{
      containerPort = 8080
      protocol      = "tcp"
    }]

    # Spring Boot Actuator health check
    healthCheck = {
      command     = ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]
      interval    = 30
      timeout     = 10
      retries     = 3
      startPeriod = 120  # Spring Boot takes time to start
    }

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/myapp-spring-boot"
        "awslogs-region"        = var.region
        "awslogs-stream-prefix" = "spring-boot"
      }
    }
  }])
}
```

## RDS PostgreSQL

```hcl
resource "aws_db_instance" "app" {
  identifier           = "myapp-${var.environment}-postgres"
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.t3.medium"
  allocated_storage    = 20
  storage_encrypted    = true

  db_name  = "myapp"
  username = "myapp"

  password_wo         = var.db_password
  password_wo_version = var.db_password_version

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.app.name

  backup_retention_period = 7
  multi_az                = var.environment == "prod"

  lifecycle {
    prevent_destroy = true
  }
}
```

## ECS Service

```hcl
resource "aws_ecs_service" "spring_boot" {
  name            = "myapp-spring-boot"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.spring_boot.arn
  desired_count   = var.environment == "prod" ? 3 : 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = module.vpc.private_subnets
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.spring_boot.arn
    container_name   = "spring-boot-app"
    container_port   = 8080
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }

  # Allow 3 minutes for Spring Boot startup before health checks kill it
  health_check_grace_period_seconds = 180
}
```

## Target Group with Spring Boot Health Path

```hcl
resource "aws_lb_target_group" "spring_boot" {
  name        = "myapp-spring-boot-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = module.vpc.vpc_id
  target_type = "ip"

  health_check {
    path                = "/actuator/health"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 10
    interval            = 30
    matcher             = "200"
  }
}
```

## Summary

Deploying Spring Boot on ECS Fargate requires careful JVM memory configuration — use `-XX:+UseContainerSupport` and `-XX:MaxRAMPercentage=75.0` to let the JVM respect container memory limits. Set `health_check_grace_period_seconds` to at least 120 seconds to give Spring Boot time to fully start before ALB health checks begin. Use Spring Boot Actuator's `/actuator/health` endpoint for health checks, and store all database credentials in Secrets Manager with environment variable injection.
