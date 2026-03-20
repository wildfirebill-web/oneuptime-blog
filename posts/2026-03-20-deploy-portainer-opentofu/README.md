# How to Deploy Portainer with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Portainer, Docker, Container Management, Self-Hosted

Description: Learn how to deploy Portainer container management UI on AWS using OpenTofu with ECS Fargate, EFS persistent storage, and ALB for managing Docker environments.

## Introduction

Portainer provides a web-based UI for managing Docker, Kubernetes, and Docker Swarm environments. This guide deploys Portainer Business Edition (or Community Edition) on AWS ECS Fargate with EFS for persistent data and ALB for HTTPS access.

## EFS for Portainer Data

```hcl
resource "aws_efs_file_system" "portainer" {
  creation_token  = "portainer-${var.environment}"
  encrypted       = true
  kms_key_id      = aws_kms_key.efs.arn
  throughput_mode = "elastic"

  tags = { Name = "portainer-data-${var.environment}" }
}

resource "aws_efs_mount_target" "portainer" {
  for_each        = toset(var.private_subnet_ids)
  file_system_id  = aws_efs_file_system.portainer.id
  subnet_id       = each.value
  security_groups = [aws_security_group.efs.id]
}

resource "aws_efs_access_point" "portainer" {
  file_system_id = aws_efs_file_system.portainer.id

  posix_user {
    uid = 1000
    gid = 1000
  }

  root_directory {
    path = "/portainer"
    creation_info {
      owner_uid   = 1000
      owner_gid   = 1000
      permissions = "755"
    }
  }
}
```

## ECS Task Definition

```hcl
resource "aws_ecs_task_definition" "portainer" {
  family                   = "portainer-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "512"
  memory                   = "1024"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  volume {
    name = "portainer-data"
    efs_volume_configuration {
      file_system_id     = aws_efs_file_system.portainer.id
      access_point_id    = aws_efs_access_point.portainer.id
      transit_encryption = "ENABLED"
      root_directory     = "/"
    }
  }

  container_definitions = jsonencode([{
    name  = "portainer"
    image = "portainer/portainer-ce:latest"

    command = [
      "--http-disabled",
      "--admin-password-file", "/run/secrets/portainer-admin-password"
    ]

    portMappings = [
      { containerPort = 9443, protocol = "tcp", name = "https" },
      { containerPort = 8000, protocol = "tcp", name = "edge" },  # Edge agent tunnel
    ]

    mountPoints = [{
      sourceVolume  = "portainer-data"
      containerPath = "/data"
      readOnly      = false
    }]

    secrets = [
      { name = "ADMIN_PASSWORD", valueFrom = aws_secretsmanager_secret.portainer_admin.arn }
    ]

    healthCheck = {
      command     = ["CMD-SHELL", "wget --no-verbose --tries=1 --spider https://localhost:9443/ || exit 1"]
      interval    = 30
      timeout     = 5
      retries     = 3
      startPeriod = 60
    }

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/portainer-${var.environment}"
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "portainer"
      }
    }
  }])
}
```

## ALB with HTTPS

```hcl
resource "aws_lb_target_group" "portainer" {
  name        = "portainer-${var.environment}"
  port        = 9443
  protocol    = "HTTPS"  # Portainer serves HTTPS directly
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    protocol            = "HTTPS"
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    interval            = 30
    matcher             = "200,302"
  }
}

resource "aws_lb_listener" "portainer_https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.portainer.arn
  }
}
```

## Admin Password Secret

```hcl
resource "random_password" "portainer_admin" {
  length   = 24
  special  = true
}

# Portainer expects bcrypt hash for admin password

resource "aws_secretsmanager_secret" "portainer_admin" {
  name = "/portainer/${var.environment}/admin-password"
}

resource "aws_secretsmanager_secret_version" "portainer_admin" {
  secret_id     = aws_secretsmanager_secret.portainer_admin.id
  secret_string = random_password.portainer_admin.result
}
```

## Route53 Record

```hcl
resource "aws_route53_record" "portainer" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "portainer.${var.domain_name}"
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```

## Conclusion

Deploying Portainer with OpenTofu provides a centralized management interface for Docker and Kubernetes environments. EFS ensures Portainer's configuration, user accounts, and environment connections persist across task replacements. For managing multiple remote Docker environments, configure the Portainer Edge Agent on remote hosts and connect them via the Edge tunnel port (8000). Disable HTTP access (`--http-disabled`) in production and enforce HTTPS-only connections.
