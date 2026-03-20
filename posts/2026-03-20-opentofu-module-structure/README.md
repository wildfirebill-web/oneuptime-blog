---
title: "OpenTofu Module Structure and File Layout"
author: nawazdhandala
tags: opentofu, terraform, iac, modules
description: "Learn the standard file structure for OpenTofu modules including main.tf, variables.tf, outputs.tf, and versions.tf."
---

# OpenTofu Module Structure and File Layout

A well-structured module is predictable and easy to navigate. OpenTofu doesn't enforce a specific file layout, but the community has converged on a standard structure that makes modules easier to understand, test, and maintain.

## Standard Module Structure

```
terraform-aws-webapp/
├── main.tf          # Primary resource definitions
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── versions.tf      # Required providers and OpenTofu version
├── README.md        # Module documentation
├── locals.tf        # Local value computations (optional)
├── data.tf          # Data source definitions (optional)
└── examples/
    ├── basic/
    │   ├── main.tf
    │   └── README.md
    └── advanced/
        ├── main.tf
        ├── variables.tf
        └── README.md
```

## main.tf — Resource Definitions

```hcl
# main.tf — contains the core resources this module creates

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.default_tags, {
    Name = "${var.name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 4, count.index)
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = merge(local.default_tags, {
    Name = "${var.name}-public-${count.index + 1}"
    Tier = "public"
  })
}
```

## variables.tf — Input Declarations

```hcl
# variables.tf — all input variables with descriptions

variable "name" {
  description = "Name prefix for all resources created by this module"
  type        = string
}

variable "vpc_cidr" {
  description = "IPv4 CIDR block for the VPC (e.g., 10.0.0.0/16)"
  type        = string
  default     = "10.0.0.0/16"

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid CIDR block."
  }
}

variable "availability_zones" {
  description = "List of availability zones to create subnets in"
  type        = list(string)
}

variable "tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

## outputs.tf — Output Values

```hcl
# outputs.tf — expose useful values to callers

output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "IDs of the public subnets"
  value       = aws_subnet.public[*].id
}
```

## versions.tf — Provider Requirements

```hcl
# versions.tf — specify required OpenTofu and provider versions

terraform {
  required_version = ">= 1.6"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

## locals.tf — Computed Values

```hcl
# locals.tf — internal computations

locals {
  default_tags = merge(
    {
      ManagedBy  = "opentofu"
      ModuleName = "vpc"
    },
    var.tags
  )
}
```

## Conclusion

The standard module structure with separate files for resources, variables, outputs, and versions makes modules easy to navigate. Consistent structure means anyone familiar with OpenTofu can find what they need without reading the entire codebase. Use the optional `locals.tf` and `data.tf` files when your module grows to keep each file focused on a single concern.
