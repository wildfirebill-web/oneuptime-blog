# How to Provision MySQL with Terraform on AWS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Terraform, AWS, RDS, Infrastructure as Code

Description: Provision a production-ready MySQL RDS instance on AWS using Terraform, including networking, parameter groups, and automated backups.

---

## Why Use Terraform for AWS RDS MySQL

Manually creating RDS instances through the AWS console does not scale and creates configuration drift between environments. Terraform lets you declare your entire MySQL infrastructure - VPC networking, security groups, parameter groups, and the RDS instance itself - as code that can be version-controlled, reviewed, and reused across dev, staging, and production.

## Project Setup

```bash
mkdir mysql-aws-terraform && cd mysql-aws-terraform
```

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  backend "s3" {
    bucket = "my-tfstate-bucket"
    key    = "rds-mysql/terraform.tfstate"
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}
```

## Networking Resources

```hcl
# network.tf
resource "aws_db_subnet_group" "mysql" {
  name        = "${var.environment}-mysql"
  description = "MySQL RDS subnet group for ${var.environment}"
  subnet_ids  = var.private_subnet_ids

  tags = local.common_tags
}

resource "aws_security_group" "mysql" {
  name        = "${var.environment}-mysql-sg"
  description = "MySQL RDS security group"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = var.app_security_group_ids
    description     = "Allow MySQL from application servers"
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = local.common_tags
}
```

## Parameter Group

```hcl
# parameters.tf
resource "aws_db_parameter_group" "mysql" {
  name        = "${var.environment}-mysql-8-0"
  family      = "mysql8.0"
  description = "Custom MySQL 8.0 parameter group"

  parameter {
    name  = "slow_query_log"
    value = "1"
  }

  parameter {
    name  = "long_query_time"
    value = "1"
  }

  parameter {
    name  = "max_connections"
    value = "500"
  }

  parameter {
    name         = "innodb_buffer_pool_size"
    value        = "{DBInstanceClassMemory*3/4}"
    apply_method = "pending-reboot"
  }

  tags = local.common_tags
}
```

## RDS Instance

```hcl
# main.tf
resource "aws_db_instance" "mysql" {
  identifier             = "${var.environment}-mysql"
  engine                 = "mysql"
  engine_version         = "8.0.35"
  instance_class         = var.instance_class
  allocated_storage      = var.allocated_storage
  max_allocated_storage  = var.max_allocated_storage
  storage_type           = "gp3"
  storage_encrypted      = true
  kms_key_id             = var.kms_key_arn

  db_name  = var.database_name
  username = var.master_username
  password = var.master_password

  db_subnet_group_name    = aws_db_subnet_group.mysql.name
  vpc_security_group_ids  = [aws_security_group.mysql.id]
  parameter_group_name    = aws_db_parameter_group.mysql.name

  backup_retention_period    = 7
  backup_window              = "02:00-03:00"
  maintenance_window         = "sun:04:00-sun:05:00"
  auto_minor_version_upgrade = true
  deletion_protection        = true
  skip_final_snapshot        = false
  final_snapshot_identifier  = "${var.environment}-mysql-final-${formatdate("YYYYMMDD", timestamp())}"

  enabled_cloudwatch_logs_exports = ["error", "slowquery"]
  monitoring_interval             = 60
  monitoring_role_arn             = aws_iam_role.rds_monitoring.arn

  tags = local.common_tags
}
```

## Outputs

```hcl
# outputs.tf
output "mysql_endpoint" {
  description = "RDS MySQL endpoint"
  value       = aws_db_instance.mysql.endpoint
}

output "mysql_port" {
  description = "RDS MySQL port"
  value       = aws_db_instance.mysql.port
}
```

## Deploying

```bash
terraform init
terraform plan -var-file=environments/production.tfvars
terraform apply -var-file=environments/production.tfvars
```

## Summary

Provisioning MySQL on AWS with Terraform involves declaring networking resources (subnet groups, security groups), a custom parameter group with tuning settings, and the RDS instance with backup and monitoring configuration. Using environment-specific variable files and an S3 backend ensures your infrastructure is reproducible, auditable, and safely managed across multiple environments.
