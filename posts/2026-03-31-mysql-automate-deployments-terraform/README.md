# How to Automate MySQL Deployments with Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Terraform, Infrastructure as Code, DevOps, Deployment

Description: Use Terraform to provision and manage MySQL infrastructure as code, enabling reproducible deployments across cloud and on-premises environments.

---

## Infrastructure as Code for MySQL

Terraform lets you declare your MySQL infrastructure in HCL (HashiCorp Configuration Language) and provision it consistently across environments. Instead of clicking through cloud consoles or running ad-hoc scripts, your database infrastructure becomes version-controlled, peer-reviewed, and reproducible.

## Project Structure

Organize your Terraform project for MySQL:

```text
mysql-terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── modules/
│   └── mysql/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev.tfvars
    └── production.tfvars
```

## Versions and Provider Configuration

```hcl
# versions.tf
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    mysql = {
      source  = "petoju/mysql"
      version = "~> 3.0"
    }
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket = "my-terraform-state"
    key    = "mysql/terraform.tfstate"
    region = "us-east-1"
  }
}
```

## Provisioning RDS MySQL

```hcl
# main.tf
resource "aws_db_subnet_group" "mysql" {
  name       = "${var.environment}-mysql-subnet-group"
  subnet_ids = var.private_subnet_ids

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}

resource "aws_db_instance" "mysql_primary" {
  identifier             = "${var.environment}-mysql-primary"
  engine                 = "mysql"
  engine_version         = "8.0.35"
  instance_class         = var.instance_class
  allocated_storage      = 100
  max_allocated_storage  = 1000
  storage_type           = "gp3"
  storage_encrypted      = true

  db_name  = var.database_name
  username = var.master_username
  password = var.master_password

  db_subnet_group_name   = aws_db_subnet_group.mysql.name
  vpc_security_group_ids = [aws_security_group.mysql.id]

  backup_retention_period   = 7
  backup_window             = "02:00-03:00"
  maintenance_window        = "sun:04:00-sun:05:00"
  auto_minor_version_upgrade = true
  deletion_protection       = true

  parameter_group_name = aws_db_parameter_group.mysql.name

  tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
  }
}
```

## Managing MySQL Resources with Terraform

Use the MySQL provider to manage databases and users:

```hcl
provider "mysql" {
  endpoint = aws_db_instance.mysql_primary.endpoint
  username = var.master_username
  password = var.master_password
}

resource "mysql_database" "app" {
  name                  = "myapp"
  default_character_set = "utf8mb4"
  default_collation     = "utf8mb4_unicode_ci"
}

resource "mysql_user" "app_user" {
  user               = "app_user"
  host               = "%"
  plaintext_password = var.app_db_password
}

resource "mysql_grant" "app_user_grant" {
  user       = mysql_user.app_user.user
  host       = mysql_user.app_user.host
  database   = mysql_database.app.name
  privileges = ["SELECT", "INSERT", "UPDATE", "DELETE"]
}
```

## Variables and Environment Configuration

```hcl
# variables.tf
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, production)"
}

variable "instance_class" {
  type        = string
  default     = "db.t3.medium"
}
```

```hcl
# environments/production.tfvars
environment    = "production"
instance_class = "db.r6g.xlarge"
```

## Deploying

```bash
# Initialize and plan
terraform init
terraform plan -var-file=environments/production.tfvars -out=plan.tfplan

# Apply the plan
terraform apply plan.tfplan

# Destroy (use with caution)
terraform destroy -var-file=environments/production.tfvars
```

## Summary

Terraform enables reproducible MySQL infrastructure deployments by codifying instance configuration, networking, parameter groups, and user management in HCL. Using the AWS RDS provider alongside the MySQL provider allows you to provision the database instance and manage its internal resources in a single pipeline. Environment-specific variable files ensure consistent promotion from dev to production.
