# How to Pass Variables to Child Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Module, Variable, Infrastructure as Code, DevOps

Description: A guide to passing variables from parent modules to child modules in OpenTofu for modular infrastructure design.

## Introduction

OpenTofu modules are self-contained, reusable infrastructure components. Passing variables to child modules allows you to customize module behavior for different environments and use cases. This guide covers the patterns and best practices for module variable passing.

## Basic Module Variable Passing

```hcl
# root/main.tf - Parent module calling child module

module "networking" {
  source = "./modules/networking"

  # Pass variables to the child module
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]
  environment        = var.environment
  project_name       = var.project_name
}
```

```hcl
# modules/networking/variables.tf - Child module variable declarations
variable "vpc_cidr" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  type        = list(string)
  description = "List of availability zones"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
}

variable "project_name" {
  type        = string
  description = "Project name for resource naming"
}
```

## Passing Root Variables to Modules

```hcl
# variables.tf - Root module variables
variable "environment" {
  type    = string
  default = "dev"
}

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

variable "project_name" {
  type = string
}

# main.tf - Pass root variables to modules
module "vpc" {
  source = "./modules/vpc"

  # Pass root-level variables
  environment  = var.environment   # var.xxx references root variables
  region       = var.aws_region
  project_name = var.project_name
}

module "compute" {
  source = "./modules/compute"

  environment  = var.environment
  project_name = var.project_name
  vpc_id       = module.vpc.vpc_id        # Pass output from another module
  subnet_ids   = module.vpc.subnet_ids    # Chain module outputs to inputs
}
```

## Using Module Outputs as Inputs

```hcl
# Chain module outputs as inputs to other modules
module "network" {
  source = "./modules/network"

  cidr_block   = "10.0.0.0/16"
  environment  = var.environment
}

module "security" {
  source = "./modules/security"

  vpc_id       = module.network.vpc_id        # Output from network module
  environment  = var.environment
}

module "database" {
  source = "./modules/rds"

  vpc_id       = module.network.vpc_id
  subnet_ids   = module.network.private_subnet_ids
  security_group_ids = [module.security.db_sg_id]
  environment  = var.environment
}
```

## Passing Complex Objects to Modules

```hcl
# root/main.tf
module "application" {
  source = "./modules/app"

  # Pass object configuration
  server_config = {
    instance_type  = var.environment == "prod" ? "t3.large" : "t3.micro"
    count          = var.environment == "prod" ? 3 : 1
    monitoring     = var.environment == "prod"
  }

  # Pass computed locals
  tags = local.common_tags

  # Pass sensitive variable
  database_password = var.database_password  # sensitive propagates
}
```

## Module Variable Defaults vs Required

```hcl
# modules/alb/variables.tf

# Required (no default) - caller must provide
variable "vpc_id" {
  type        = string
  description = "VPC ID where the ALB will be created"
}

variable "subnet_ids" {
  type        = list(string)
  description = "Subnet IDs for ALB"
}

# Optional (has default) - caller can override
variable "internal" {
  type        = bool
  description = "Whether ALB should be internal or internet-facing"
  default     = false
}

variable "idle_timeout" {
  type        = number
  description = "ALB connection idle timeout in seconds"
  default     = 60
}

variable "enable_deletion_protection" {
  type        = bool
  description = "Prevent accidental deletion of the ALB"
  default     = false
}
```

## Conclusion

Passing variables to child modules is the foundation of modular infrastructure design in OpenTofu. Using root-level variables with descriptive names, chaining module outputs as inputs to dependent modules, and passing computed locals create a clean, readable infrastructure composition. Always document required vs optional variables in child modules and use appropriate defaults that work for most use cases while allowing override when needed.
