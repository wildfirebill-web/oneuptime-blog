# How to Organize Files in an OpenTofu Project

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Project Organization, Best Practices, Infrastructure as Code, DevOps

Description: Best practices and patterns for organizing files and directories in OpenTofu infrastructure projects.

## Introduction

A well-organized OpenTofu project is easier to understand, maintain, and scale. This guide presents proven patterns for organizing your OpenTofu files and directories, from simple single-environment projects to complex multi-environment infrastructures.

## Pattern 1: Simple Single-Environment Project

```text
infrastructure/
├── versions.tf          # Provider and version requirements
├── main.tf              # Core resources
├── variables.tf         # Variable declarations
├── outputs.tf           # Output declarations
├── locals.tf            # Local value computations
├── data.tf              # Data source definitions
├── terraform.tfvars     # Variable values
├── .terraform.lock.hcl  # Lock file (commit this)
├── .gitignore
└── README.md
```

## Pattern 2: Feature-Based Organization

Group files by infrastructure feature:

```text
infrastructure/
├── versions.tf
├── variables.tf
├── outputs.tf
├── locals.tf
├── networking.tf        # VPC, subnets, routing
├── security.tf          # Security groups, NACLs, IAM
├── compute.tf           # EC2, ASG, Lambda
├── database.tf          # RDS, DynamoDB
├── storage.tf           # S3, EFS
├── monitoring.tf        # CloudWatch, alerts
└── dns.tf               # Route53 records
```

## Pattern 3: Multi-Environment with Workspaces

```text
infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
├── locals.tf
├── dev.tfvars            # Dev environment values
├── staging.tfvars        # Staging values
├── prod.tfvars           # Production values
└── backends/
    ├── dev.hcl           # Dev backend config
    ├── staging.hcl       # Staging backend config
    └── prod.hcl          # Prod backend config
```

```bash
# Usage:

tofu init -backend-config=backends/prod.hcl
tofu plan -var-file=prod.tfvars
```

## Pattern 4: Environment-Specific Directories

```text
infrastructure/
├── modules/              # Reusable modules
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── README.md
│   └── eks-cluster/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── environments/
    ├── dev/
    │   ├── main.tf       # Uses modules
    │   ├── versions.tf
    │   ├── variables.tf
    │   └── terraform.tfvars
    ├── staging/
    │   ├── main.tf
    │   ├── versions.tf
    │   └── terraform.tfvars
    └── prod/
        ├── main.tf
        ├── versions.tf
        └── terraform.tfvars
```

## Module Structure Best Practices

```hcl
# modules/vpc/main.tf
resource "aws_vpc" "this" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames

  tags = merge(
    var.tags,
    {
      Name = "${var.name}-vpc"
    }
  )
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnets)
  vpc_id            = aws_vpc.this.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
}
```

```hcl
# modules/vpc/variables.tf
variable "name" {
  type        = string
  description = "Name prefix for VPC resources"
}

variable "cidr_block" {
  type        = string
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

variable "public_subnets" {
  type        = list(string)
  description = "List of CIDR blocks for public subnets"
  default     = []
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  value       = aws_vpc.this.id
  description = "The ID of the VPC"
}

output "public_subnet_ids" {
  value       = aws_subnet.public[*].id
  description = "IDs of public subnets"
}
```

## The .gitignore File

```gitignore
# .gitignore for OpenTofu projects
.terraform/
.terraform.tfstate
.terraform.tfstate.backup
*.tfstate
*.tfstate.backup
*.tfplan
*.auto.tfvars
!example.auto.tfvars   # Keep examples
.terraformrc
terraform.rc
*.sensitive.tfvars     # Sensitive variable files
crash.log
```

## Documentation Standards

```hcl
# Every module should have a versions.tf with clear documentation
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0, < 6.0"
    }
  }
}
```

```markdown
# README.md for each module
## Module: VPC

Creates a VPC with public and private subnets.

### Usage
...

### Inputs
| Name | Type | Default | Description |
|------|------|---------|-------------|

### Outputs
| Name | Description |
|------|-------------|
```

## Conclusion

Organizing OpenTofu files by concern (networking, compute, security) within a consistent directory structure makes infrastructure projects manageable at any scale. Using modules for reusable components and separating environment configurations into distinct directories or variable files provides the flexibility to manage complex multi-environment deployments effectively.
