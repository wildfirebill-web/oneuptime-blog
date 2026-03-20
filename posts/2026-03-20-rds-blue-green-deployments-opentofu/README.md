# How to Set Up RDS Blue-Green Deployments with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, RDS, Blue-Green Deployment, Database, Infrastructure as Code

Description: Learn how to create and manage AWS RDS Blue/Green deployments for zero-downtime database schema changes and engine upgrades using OpenTofu.

## Introduction

AWS RDS Blue/Green Deployments allow you to make database changes in a staging (green) environment that mirrors production (blue), test them, and then perform a fast switchover with minimal downtime. OpenTofu manages the blue/green deployment lifecycle.

## Creating the Blue/Green Deployment

```hcl
# First, the production (blue) database must exist

resource "aws_db_instance" "blue" {
  identifier        = "${var.app_name}-db-${var.environment}"
  engine            = "mysql"
  engine_version    = "8.0.35"
  instance_class    = "db.r6g.large"
  username          = var.db_username
  password          = var.db_password
  allocated_storage = 100
  storage_type      = "gp3"

  backup_retention_period    = 7
  backup_window              = "03:00-04:00"
  maintenance_window         = "Mon:04:00-Mon:05:00"

  # Blue/Green requires automated backups
  db_subnet_group_name       = aws_db_subnet_group.main.name
  vpc_security_group_ids     = [aws_security_group.db.id]
  multi_az                   = true

  # Required for Blue/Green
  parameter_group_name = aws_db_parameter_group.mysql80.name

  tags = {
    Environment = var.environment
    Role        = "blue"
    ManagedBy   = "opentofu"
  }
}

# Create the Blue/Green deployment
resource "aws_rds_blue_green_deployment" "main" {
  blue_green_deployment_name = "${var.app_name}-bluegreen"
  source                     = aws_db_instance.blue.arn

  # Optionally specify a different engine version for the green environment
  target_engine_version   = "8.0.36"
  target_db_instance_class = "db.r6g.xlarge"  # optional: test a new instance size

  timeouts {
    create = "60m"
    delete = "60m"
    update = "90m"
  }

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Parameter Group

```hcl
resource "aws_db_parameter_group" "mysql80" {
  name   = "${var.app_name}-mysql80"
  family = "mysql8.0"

  parameter {
    name  = "binlog_format"
    value = "ROW"  # required for Blue/Green replication
  }

  parameter {
    name  = "binlog_row_image"
    value = "Full"
  }
}
```

## Switchover Script

After verifying the green environment, perform the switchover.

```bash
#!/bin/bash
# scripts/bluegreen-switchover.sh

DEPLOYMENT_NAME=$(tofu output -raw bluegreen_deployment_name)

echo "Starting switchover for deployment: $DEPLOYMENT_NAME"

aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier "$DEPLOYMENT_NAME" \
  --switchover-timeout 300 \
  --region us-east-1

echo "Waiting for switchover to complete..."
aws rds wait db-instance-available \
  --db-instance-identifier "${APP_NAME}-db-${ENVIRONMENT}"

echo "Switchover complete!"
```

## Cleanup After Switchover

```bash
# After successful switchover, delete the old blue instance
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier "$DEPLOYMENT_NAME" \
  --delete-source
```

## Outputs

```hcl
output "bluegreen_deployment_name" {
  value = aws_rds_blue_green_deployment.main.blue_green_deployment_name
}

output "green_db_endpoint" {
  description = "Endpoint of the green (staging) database"
  value       = aws_rds_blue_green_deployment.main.id
}
```

## Deploying

```bash
tofu init
tofu plan -out=tfplan
tofu apply tfplan
```

## Summary

RDS Blue/Green Deployments enable safe, low-downtime database changes by maintaining a synchronized green environment. OpenTofu manages deployment creation and the production database configuration - while switchover is performed via the AWS CLI or console when ready.
