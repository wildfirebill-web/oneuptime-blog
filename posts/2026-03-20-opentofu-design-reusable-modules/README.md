# How to Design Reusable Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules

Description: Learn the principles and patterns for designing reusable, composable OpenTofu modules that work across teams, environments, and cloud accounts.

## Introduction

A reusable module solves one problem well, accepts sensible defaults, and exposes the right outputs. Modules designed with reusability in mind can be consumed across different environments, teams, and even organizations without modification.

## Single Responsibility

Each module should do one thing:

```text
Good module boundaries:
├── modules/vpc           - Networking only
├── modules/eks           - Kubernetes cluster only
├── modules/rds           - Database only
└── modules/app-tier      - Compute + load balancer (composes the above)

Avoid:
└── modules/entire-stack  - Creates everything - hard to reuse
```

## Use Variables for Everything That Changes

```hcl
# variables.tf - expose knobs for every environment-specific value

variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type for the application servers"
  default     = "t3.small"
}

variable "min_capacity" {
  type        = number
  description = "Minimum number of instances in the auto-scaling group"
  default     = 1
}

variable "max_capacity" {
  type        = number
  description = "Maximum number of instances in the auto-scaling group"
  default     = 10
}
```

## Provide Sensible Defaults

Defaults reduce the caller's configuration burden for common cases:

```hcl
variable "enable_deletion_protection" {
  type        = bool
  description = "Enable deletion protection on the database"
  default     = true  # Safe default - callers opt out explicitly
}

variable "backup_retention_days" {
  type        = number
  description = "Days to retain automated backups"
  default     = 7
}
```

## Expose Useful Outputs

Callers need to reference your module's resources:

```hcl
# outputs.tf - expose everything a consumer might need
output "vpc_id" {
  description = "The ID of the created VPC"
  value       = aws_vpc.this.id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}
```

## Use tags Variable for Consistency

Accept a tags map so callers can apply organizational tagging standards:

```hcl
variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources"
  default     = {}
}

locals {
  common_tags = merge({
    Module      = "vpc"
    ManagedBy   = "opentofu"
  }, var.tags)
}

resource "aws_vpc" "this" {
  cidr_block = var.cidr_block
  tags       = merge(local.common_tags, { Name = var.name })
}
```

## Include an Examples Directory

```text
terraform-aws-vpc/
├── main.tf
├── variables.tf
├── outputs.tf
└── examples/
    ├── basic/
    │   └── main.tf   - Minimal usage example
    └── complete/
        └── main.tf   - Full-featured example
```

```hcl
# examples/basic/main.tf
module "vpc" {
  source = "../../"

  name       = "example-vpc"
  cidr_block = "10.0.0.0/16"
}
```

## Avoid Hard-Coded Values

```hcl
# Bad - hard-coded region and account
resource "aws_iam_role" "this" {
  assume_role_policy = jsonencode({
    Statement = [{
      Principal = { Service = "ec2.amazonaws.com" }
    }]
  })
}

# Good - use data sources and variables
data "aws_partition" "current" {}

resource "aws_iam_role" "this" {
  assume_role_policy = jsonencode({
    Statement = [{
      Principal = { Service = "ec2.${data.aws_partition.current.dns_suffix}" }
    }]
  })
}
```

## Conclusion

Reusable modules have clear boundaries, parameterize all variable values, provide safe defaults, and expose comprehensive outputs. Structure each module around a single concern, ship examples alongside the module, and avoid hard-coded values that tie the module to a specific account or region. These practices allow a single module to serve dozens of teams without modification.
