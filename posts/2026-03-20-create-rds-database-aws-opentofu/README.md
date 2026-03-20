# How to Create an RDS Database with OpenTofu on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Database, Infrastructure as Code

Description: Learn how to create a production-ready AWS RDS database instance with OpenTofu, including Multi-AZ, encryption, automated backups, and parameter groups.

## Introduction

Amazon RDS manages relational databases so you don't have to worry about patching, backups, or failover. This guide shows how to create a PostgreSQL RDS instance with OpenTofu that is ready for production workloads.

## Subnet Group

RDS requires a subnet group spanning at least two AZs:

```hcl
resource "aws_db_subnet_group" "main" {
  name        = "${var.name}-db-subnet-group"
  description = "Subnet group for ${var.name} RDS instance"
  subnet_ids  = var.private_subnet_ids

  tags = {
    Name = "${var.name}-db-subnet-group"
  }
}
```

## Security Group

```hcl
resource "aws_security_group" "rds" {
  name        = "${var.name}-rds-sg"
  description = "Security group for RDS instance"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.allowed_security_group_ids
    description     = "PostgreSQL from application tier"
  }
}
```

## Parameter Group

```hcl
resource "aws_db_parameter_group" "postgres" {
  name        = "${var.name}-postgres16"
  family      = "postgres16"
  description = "Custom parameter group for ${var.name}"

  parameter {
    name  = "log_connections"
    value = "1"
  }

  parameter {
    name  = "log_disconnections"
    value = "1"
  }

  parameter {
    name         = "shared_preload_libraries"
    value        = "pg_stat_statements"
    apply_method = "pending-reboot"
  }
}
```

## RDS Instance

```hcl
resource "random_password" "db" {
  length           = 32
  special          = true
  override_special = "!#$%&*()-_=+[]{}<>:?"
}

resource "aws_db_instance" "main" {
  identifier = "${var.name}-db"

  # Engine configuration
  engine               = "postgres"
  engine_version       = "16.3"
  instance_class       = var.db_instance_class
  parameter_group_name = aws_db_parameter_group.postgres.name

  # Storage
  storage_type          = "gp3"
  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.allocated_storage * 2  # Enable storage autoscaling
  storage_encrypted     = true
  kms_key_id            = var.kms_key_arn

  # Credentials
  db_name  = var.db_name
  username = var.db_username
  password = random_password.db.result
  # Store password in Secrets Manager (see manage_master_user_password below)
  manage_master_user_password = false  # Set true to use Secrets Manager

  # Networking
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]
  publicly_accessible    = false

  # High availability
  multi_az = var.environment == "prod" ? true : false

  # Backups
  backup_retention_period   = 7
  backup_window             = "03:00-04:00"
  maintenance_window        = "sun:04:00-sun:05:00"
  copy_tags_to_snapshot     = true
  delete_automated_backups  = true

  # Monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]

  # Deletion protection
  deletion_protection = var.environment == "prod" ? true : false
  skip_final_snapshot = var.environment == "prod" ? false : true
  final_snapshot_identifier = var.environment == "prod" ? "${var.name}-final-snapshot" : null

  tags = {
    Name        = "${var.name}-db"
    Environment = var.environment
  }
}
```

## Store the Password in AWS Secrets Manager

```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name        = "${var.name}/db-password"
  description = "RDS master password for ${var.name}"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  secret_string = jsonencode({
    username = var.db_username
    password = random_password.db.result
    host     = aws_db_instance.main.address
    port     = aws_db_instance.main.port
    dbname   = var.db_name
  })
}
```

## Outputs

```hcl
output "db_endpoint"    { value = aws_db_instance.main.address }
output "db_port"        { value = aws_db_instance.main.port }
output "db_name"        { value = aws_db_instance.main.db_name }
output "secret_arn"     { value = aws_secretsmanager_secret.db_password.arn }
```

## Conclusion

A production RDS instance needs encryption, Multi-AZ, backups, and enhanced monitoring from day one. Store credentials in Secrets Manager rather than passing them as plain-text outputs, and enable deletion protection in production to prevent accidental data loss.
