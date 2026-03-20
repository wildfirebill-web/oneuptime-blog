# How to Use Consistent File Layout in OpenTofu Projects

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, File Layout, Best Practices, Infrastructure as Code, Organizations, Team Collaboration

Description: Learn the standard file layout for OpenTofu configurations and modules - what goes in each file and why - so every team member knows exactly where to find and add code.

## Introduction

A consistent file layout means every OpenTofu practitioner on your team knows exactly where variable declarations live, where to add a new resource, and where outputs are defined - without reading the entire configuration. This guide defines the standard layout and explains the rationale behind each file.

## Standard Root Configuration Layout

```text
environment/prod/
├── main.tf          # Resource declarations and module calls
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── locals.tf        # Local value computations
├── data.tf          # Data source declarations
├── providers.tf     # Provider and required_providers blocks
├── backend.tf       # Backend configuration
├── versions.tf      # required_version and required_providers (alternative to providers.tf)
└── terraform.tfvars # Variable values (committed if non-sensitive)
```

## What Goes in Each File

### main.tf

```hcl
# main.tf - resource blocks and module calls

# The primary file - where the infrastructure is defined

module "networking" {
  source      = "../../modules/networking"
  vpc_cidr    = var.vpc_cidr
  environment = local.environment
}

resource "aws_ecs_cluster" "main" {
  name = "${local.name_prefix}-cluster"
  tags = local.common_tags
}
```

### variables.tf

```hcl
# variables.tf - ONLY variable declarations, no default values with secrets
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
}
```

### locals.tf

```hcl
# locals.tf - computed values derived from variables
locals {
  environment = var.environment
  name_prefix = "${var.company}-${var.environment}"
  common_tags = {
    ManagedBy   = "opentofu"
    Environment = var.environment
    Team        = var.team
  }
}
```

### data.tf

```hcl
# data.tf - all data sources in one place
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

### outputs.tf

```hcl
# outputs.tf - all outputs in one place
output "vpc_id" {
  description = "ID of the VPC"
  value       = module.networking.vpc_id
}

output "cluster_arn" {
  description = "ARN of the ECS cluster"
  value       = aws_ecs_cluster.main.arn
}
```

## Module File Layout

```text
modules/networking/
├── main.tf       # Resource definitions
├── variables.tf  # Input variable declarations
├── outputs.tf    # Output declarations
└── README.md     # Module documentation (required for registry)
```

## terraform.tfvars vs variables.tf

```hcl
# variables.tf - DECLARATIONS (committed to version control)
variable "instance_type" {
  type    = string
  default = "t3.medium"
}

# terraform.tfvars - VALUES (committed if non-sensitive)
instance_type = "t3.medium"
vpc_cidr      = "10.0.0.0/16"
environment   = "prod"
```

```bash
# .gitignore - exclude files with real secrets
*.tfvars
!example.tfvars    # Commit only the example with placeholder values
```

## Conclusion

Standard file layout is a team convention that pays dividends every code review. When every engineer knows that data sources live in `data.tf`, variables are declared in `variables.tf`, and computed values are in `locals.tf`, navigation becomes effortless. Enforce this layout through code review guidelines and `tflint` rules that flag misplaced declarations.
