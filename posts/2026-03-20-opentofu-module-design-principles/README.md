# How to Explain OpenTofu Module Design Principles

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Modules, Design Principles, Best Practices, Infrastructure as Code

Description: Learn the core principles for designing reusable, maintainable OpenTofu modules that follow established best practices.

## Introduction

OpenTofu modules are reusable packages of infrastructure configuration. Well-designed modules are easy to use, hard to misuse, and composable with other modules. Poor module design leads to tightly coupled configurations, unclear interfaces, and difficult maintenance. This post covers the principles that distinguish good modules from bad ones.

## Principle 1: Single Responsibility

Each module should do one thing well.

```hcl
# BAD: Module does too much

# modules/everything/main.tf
# Creates: VPC, EKS cluster, RDS, S3, IAM, CloudFront...
# This is impossible to reuse in different contexts

# GOOD: Focused modules
modules/
├── vpc/           # Only networking resources
├── eks-cluster/   # Only the EKS cluster
├── rds-postgres/  # Only RDS PostgreSQL
└── s3-website/    # Only S3 static hosting
```

## Principle 2: Clear Input/Output Interface

Variables and outputs form the module's public interface - design them carefully.

```hcl
# modules/rds-postgres/variables.tf

variable "identifier" {
  type        = string
  description = "Unique identifier for the RDS instance"
}

variable "instance_class" {
  type        = string
  description = "RDS instance type"
  default     = "db.t3.medium"

  validation {
    condition     = can(regex("^db\\.", var.instance_class))
    error_message = "instance_class must be a valid RDS instance type starting with 'db.'"
  }
}

variable "allocated_storage_gb" {
  type        = number
  description = "Storage size in GB"
  default     = 20

  validation {
    condition     = var.allocated_storage_gb >= 20 && var.allocated_storage_gb <= 65536
    error_message = "Storage must be between 20 and 65536 GB"
  }
}
```

```hcl
# modules/rds-postgres/outputs.tf

output "endpoint" {
  description = "Connection endpoint for the RDS instance"
  value       = aws_db_instance.main.endpoint
}

output "port" {
  description = "Port number for the RDS instance"
  value       = aws_db_instance.main.port
}

output "instance_id" {
  description = "RDS instance identifier"
  value       = aws_db_instance.main.id
}
```

## Principle 3: Sensible Defaults with Customization

Provide secure, production-ready defaults while allowing customization.

```hcl
variable "backup_retention_days" {
  type        = number
  description = "Number of days to retain automated backups"
  default     = 7  # sensible default for most use cases

  validation {
    condition     = var.backup_retention_days >= 1 && var.backup_retention_days <= 35
    error_message = "backup_retention_days must be between 1 and 35"
  }
}

variable "deletion_protection" {
  type        = bool
  description = "Enable deletion protection (recommended for production)"
  default     = true  # safe default - explicit action required to disable
}
```

## Principle 4: No Hardcoded Values

Everything that might need to change should be a variable.

```hcl
# BAD: Hardcoded values
resource "aws_db_instance" "main" {
  instance_class = "db.t3.medium"  # hardcoded
  engine_version = "15.4"          # hardcoded
  region         = "us-east-1"     # hardcoded
}

# GOOD: Parameterized
resource "aws_db_instance" "main" {
  instance_class = var.instance_class
  engine_version = var.engine_version
  # region is set via the provider, not in the resource
}
```

## Principle 5: Composition Over Complexity

Build modules that compose well with other modules.

```hcl
# Root module composes smaller modules
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "eks" {
  source     = "./modules/eks-cluster"
  vpc_id     = module.vpc.id         # pass outputs between modules
  subnet_ids = module.vpc.private_subnet_ids
}

module "rds" {
  source     = "./modules/rds-postgres"
  vpc_id     = module.vpc.id
  subnet_ids = module.vpc.database_subnet_ids
  allowed_security_group_ids = [module.eks.node_security_group_id]
}
```

## Principle 6: Version Your Modules

Always version modules when sharing them across teams.

```hcl
# For Git-sourced modules, always pin to a tag
module "vpc" {
  source = "git::https://github.com/my-org/opentofu-modules.git//vpc?ref=v2.1.0"
}

# For registry modules, pin versions
module "vpc" {
  source  = "registry.opentofu.org/terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}
```

## Summary

Well-designed OpenTofu modules follow the single responsibility principle, expose a clear variable/output interface with validation and sensible defaults, avoid hardcoded values, and compose cleanly with other modules. Version all shared modules to prevent unexpected breaking changes when modules are updated. A good module should be easy to use correctly and difficult to use incorrectly.
