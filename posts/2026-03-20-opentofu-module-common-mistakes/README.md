---
title: "Common Mistakes When Writing OpenTofu Modules"
author: nawazdhandala
tags: opentofu, terraform, iac, modules, best-practices
description: "Avoid these common pitfalls when writing OpenTofu modules to keep your infrastructure code maintainable and reliable."
---

# Common Mistakes When Writing OpenTofu Modules

Writing OpenTofu modules seems straightforward, but many teams fall into recurring patterns that make their modules brittle, hard to use, or difficult to maintain. Here are the most common mistakes and how to fix them.

## Mistake 1: Hardcoding Provider Configuration

```hcl
# BAD: hardcoded region inside module
provider "aws" {
  region = "us-east-1"  # Never do this in a module
}

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
}
```

```hcl
# GOOD: let the caller configure the provider
# modules/vpc/main.tf — no provider block
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
}

# Root module — caller configures provider
provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source     = "./modules/vpc"
  cidr_block = "10.0.0.0/16"
}
```

## Mistake 2: No Variable Validation

```hcl
# BAD: accepting anything
variable "environment" {
  type = string
}

# GOOD: validate allowed values
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be 'dev', 'staging', or 'prod'."
  }
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string

  validation {
    condition     = can(cidrnetmask(var.cidr_block))
    error_message = "Must be a valid CIDR block."
  }
}
```

## Mistake 3: Not Using Descriptions

```hcl
# BAD: no documentation
variable "size" {}
output "id" {
  value = aws_db_instance.main.id
}

# GOOD: every variable and output documented
variable "instance_class" {
  description = "RDS instance class (e.g., db.t3.medium, db.r5.large)"
  type        = string
  default     = "db.t3.medium"
}

output "endpoint" {
  description = "Database connection endpoint (hostname:port)"
  value       = "${aws_db_instance.main.address}:${aws_db_instance.main.port}"
}
```

## Mistake 4: Exposing Too Few Outputs

```hcl
# BAD: only exposes one output — callers can't do much with it
output "id" {
  value = aws_instance.app.id
}

# GOOD: expose what callers need
output "id" {
  description = "Instance ID"
  value       = aws_instance.app.id
}

output "arn" {
  description = "Instance ARN"
  value       = aws_instance.app.arn
}

output "private_ip" {
  description = "Private IP address"
  value       = aws_instance.app.private_ip
}

output "security_group_id" {
  description = "Security group ID — use to grant access from other resources"
  value       = aws_security_group.main.id
}

output "iam_role_name" {
  description = "IAM role name — use to attach additional policies"
  value       = aws_iam_role.main.name
}
```

## Mistake 5: Creating Resources the Caller Already Has

```hcl
# BAD: always creates a VPC — what if caller has one?
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# GOOD: accept existing resources or create new ones
variable "vpc_id" {
  description = "ID of existing VPC. If null, a new VPC will be created."
  type        = string
  default     = null
}

resource "aws_vpc" "main" {
  count      = var.vpc_id == null ? 1 : 0
  cidr_block = var.vpc_cidr
}

locals {
  vpc_id = var.vpc_id != null ? var.vpc_id : aws_vpc.main[0].id
}
```

## Mistake 6: Not Handling the count/for_each Index Issue

```hcl
# BAD: using count with potentially reordering list
variable "user_names" {
  type    = list(string)
  default = ["alice", "bob", "charlie"]
}

resource "aws_iam_user" "users" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
  # Removing "bob" shifts indices — destroys alice!
}

# GOOD: use for_each with set
resource "aws_iam_user" "users" {
  for_each = toset(var.user_names)
  name     = each.key
  # Removing "bob" only destroys bob
}
```

## Mistake 7: Overly Complex Modules

```hcl
# BAD: one module that does everything
module "full_stack" {
  source = "./modules/full-stack"

  # 50+ variables...
  create_vpc      = true
  create_eks      = true
  create_rds      = true
  create_redis    = true
  create_bastion  = true
  # ...
}

# GOOD: composable single-responsibility modules
module "vpc" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}

module "eks" {
  source   = "./modules/eks"
  vpc_id   = module.vpc.vpc_id
  subnets  = module.vpc.private_subnets
}

module "rds" {
  source   = "./modules/rds"
  vpc_id   = module.vpc.vpc_id
  subnets  = module.vpc.database_subnets
}
```

## Mistake 8: Ignoring Required Versions

```hcl
# BAD: no version constraints
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

# GOOD: specify compatible version range
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"
    }
  }
}
```

## Conclusion

The most common module mistakes boil down to lack of flexibility, missing documentation, and tight coupling. Design modules for reuse from the start: validate inputs, expose comprehensive outputs, avoid hardcoded assumptions, and keep each module focused on a single concern.
