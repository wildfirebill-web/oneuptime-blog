# How to Set Up a Basic OpenTofu Project Structure

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Project Structure, Best Practices, Infrastructure as Code, DevOps

Description: A guide to organizing OpenTofu files and directories for maintainable, scalable infrastructure projects.

## Introduction

A well-organized OpenTofu project structure makes your infrastructure code more maintainable, easier to understand, and simpler to collaborate on. This guide covers standard conventions and recommended patterns for structuring OpenTofu projects.

## Minimal Project Structure

For simple projects:

```
my-infrastructure/
├── main.tf          # Main resources
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── versions.tf      # Provider and version constraints
├── terraform.tfvars # Variable values (don't commit secrets!)
└── .gitignore       # Ignore .terraform/ and sensitive files
```

## Standard Project Structure

For most team projects:

```
my-infrastructure/
├── main.tf              # Primary resources
├── variables.tf         # Input variable declarations
├── outputs.tf           # Output value declarations
├── versions.tf          # required_providers and required_version
├── locals.tf            # Local value computations
├── data.tf              # Data source definitions
├── terraform.tfvars     # Default variable values
├── dev.tfvars           # Dev environment variables
├── staging.tfvars       # Staging environment variables
├── prod.tfvars          # Production environment variables
├── .terraform.lock.hcl  # Provider version lock (commit this!)
├── .gitignore
└── README.md
```

## Multi-Module Project Structure

For complex, reusable infrastructure:

```
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── versions.tf
│   └── database/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
└── environments/
    ├── dev/
    │   ├── main.tf
    │   ├── terraform.tfvars
    │   └── backend.hcl
    ├── staging/
    │   ├── main.tf
    │   ├── terraform.tfvars
    │   └── backend.hcl
    └── prod/
        ├── main.tf
        ├── terraform.tfvars
        └── backend.hcl
```

## File Contents

### versions.tf

```hcl
# versions.tf - Always have this file
terraform {
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket         = "my-tofu-state"
    key            = "infrastructure/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "tofu-state-lock"
  }
}
```

### variables.tf

```hcl
# variables.tf - All variable declarations
variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "aws_region" {
  type        = string
  description = "AWS region to deploy resources in"
  default     = "us-east-1"
}

variable "project_name" {
  type        = string
  description = "Project name used for resource naming"
}
```

### locals.tf

```hcl
# locals.tf - Computed values used across resources
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "OpenTofu"
  }

  name_prefix = "${var.project_name}-${var.environment}"
}
```

### main.tf

```hcl
# main.tf - Primary resource definitions
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = local.common_tags
  }
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"

  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}
```

### outputs.tf

```hcl
# outputs.tf - Values to expose after apply
output "vpc_id" {
  description = "The ID of the main VPC"
  value       = aws_vpc.main.id
}
```

### .gitignore

```gitignore
# .gitignore for OpenTofu projects
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.tfstate.lock.info
*.auto.tfvars  # Be careful with auto.tfvars containing secrets
override.tf
override.tf.json
*_override.tf
*_override.tf.json
.terraformrc
terraform.rc
```

## Conclusion

A well-structured OpenTofu project separates concerns into logical files, making it easy to navigate and maintain. The standard split into `main.tf`, `variables.tf`, `outputs.tf`, and `versions.tf` is universally recognized and expected by the community. As projects grow, modularizing into reusable components under `modules/` and separating environment-specific values into `tfvars` files scales naturally.
