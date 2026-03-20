# How to Deploy Gitea with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Gitea, Git, Self-Hosted, DevOps

Description: Learn how to deploy Gitea self-hosted Git service on AWS using OpenTofu with RDS PostgreSQL backend, ECS Fargate, and ALB for a production-ready Git platform.

## Introduction

Gitea is a lightweight, self-hosted Git service with a GitHub-like interface. This guide deploys Gitea on AWS ECS Fargate with RDS PostgreSQL for persistence, EFS for repository storage, and ALB for HTTPS termination.

## RDS PostgreSQL Backend

```hcl
resource "aws_db_instance" "gitea" {
  identifier            = "gitea-${var.environment}"
  engine                = "postgres"
  engine_version        = "15.4"
  instance_class        = "db.t3.small"
  allocated_storage     = 20
  max_allocated_storage = 200
  storage_type          = "gp3"
  storage_encrypted     = true

  db_name  = "gitea"
  username = "gitea"
  password = random_password.db_password.result

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name

  backup_retention_period  = 7
  deletion_protection      = true
  skip_final_snapshot      = false
}

resource "random_password" "db_password" {
  length  = 32
  special = false  # Avoid special chars that break connection strings
}
```

## EFS for Repository Storage

```hcl
resource "aws_efs_file_system" "gitea" {
  creation_token = "gitea-repos-${var.environment}"
  encrypted      = true
  kms_key_id     = aws_kms_key.efs.arn
  throughput_mode = "elastic"

  tags = { Name = "gitea-repos" }
}

resource "aws_efs_mount_target" "gitea" {
  for_each       = toset(var.private_subnet_ids)
  file_system_id = aws_efs_file_system.gitea.id
  subnet_id      = each.value
  security_groups = [aws_security_group.efs.id]
}
```

## ECS Task Definition

```hcl
resource "aws_ecs_task_definition" "gitea" {
  family                   = "gitea-${var.environment}"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "1024"
  memory                   = "2048"
  execution_role_arn       = aws_iam_role.ecs_execution.arn

  volume {
    name = "gitea-data"
    efs_volume_configuration {
      file_system_id     = aws_efs_file_system.gitea.id
      transit_encryption = "ENABLED"
      root_directory     = "/"
    }
  }

  container_definitions = jsonencode([{
    name  = "gitea"
    image = "gitea/gitea:1.21"

    environment = [
      { name = "GITEA__database__DB_TYPE",    value = "postgres" },
      { name = "GITEA__database__HOST",       value = "${aws_db_instance.gitea.address}:5432" },
      { name = "GITEA__database__NAME",       value = "gitea" },
      { name = "GITEA__database__USER",       value = "gitea" },
      { name = "GITEA__server__DOMAIN",       value = var.gitea_hostname },
      { name = "GITEA__server__ROOT_URL",     value = "https://${var.gitea_hostname}" },
      { name = "GITEA__server__SSH_DOMAIN",   value = var.gitea_hostname },
      { name = "GITEA__server__SSH_PORT",     value = "22" },
      { name = "GITEA__service__DISABLE_REGISTRATION", value = "true" },
      { name = "GITEA__mailer__ENABLED",      value = "true" },
      { name = "GITEA__mailer__FROM",         value = "gitea@${var.domain_name}" },
      { name = "GITEA__mailer__SMTP_ADDR",    value = "email-smtp.${var.aws_region}.amazonaws.com" },
      { name = "GITEA__mailer__SMTP_PORT",    value = "587" },
    ]

    secrets = [
      { name = "GITEA__database__PASSWD",      valueFrom = aws_secretsmanager_secret.db_password.arn },
      { name = "GITEA__security__SECRET_KEY",  valueFrom = aws_secretsmanager_secret.secret_key.arn },
      { name = "GITEA__security__INTERNAL_TOKEN", valueFrom = aws_secretsmanager_secret.internal_token.arn },
    ]

    portMappings = [
      { containerPort = 3000, protocol = "tcp" },
      { containerPort = 22,   protocol = "tcp" },  # SSH for git
    ]

    mountPoints = [{
      sourceVolume  = "gitea-data"
      containerPath = "/data"
      readOnly      = false
    }]

    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = "/ecs/gitea"
        "awslogs-region"        = var.aws_region
        "awslogs-stream-prefix" = "gitea"
      }
    }
  }])
}
```

## ALB and Route53

```hcl
resource "aws_lb_target_group" "gitea" {
  name        = "gitea-${var.environment}"
  port        = 3000
  protocol    = "HTTP"
  vpc_id      = var.vpc_id
  target_type = "ip"

  health_check {
    path                = "/"
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}

resource "aws_route53_record" "gitea" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = var.gitea_hostname
  type    = "A"

  alias {
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
    evaluate_target_health = true
  }
}
```

## Conclusion

Deploying Gitea with OpenTofu provides a lightweight, self-hosted Git platform with the operational benefits of managed infrastructure. EFS provides durable repository storage that survives task replacements, and RDS handles the metadata database with automated backups. Disable public registration in production (`DISABLE_REGISTRATION = true`) and set up LDAP or OIDC authentication for team access.
