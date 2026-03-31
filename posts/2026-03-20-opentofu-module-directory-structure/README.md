# How to Structure an OpenTofu Module Directory

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn the best practices for structuring an OpenTofu module directory with proper file organization, documentation, and testing layout.

## Introduction

A well-structured OpenTofu module directory makes modules easy to understand, test, maintain, and share. This post covers the conventional file layout used by the OpenTofu community and the Terraform Registry.

## Standard Module Structure

```hcl
modules/my-module/
├── README.md              # Module documentation
├── main.tf                # Core resources
├── variables.tf           # Input variable declarations
├── outputs.tf             # Output value declarations
├── versions.tf            # Provider and OpenTofu version requirements
├── locals.tf              # Local value computations (optional)
├── data.tf                # Data source lookups (optional)
├── examples/
│   ├── basic/
│   │   ├── main.tf        # Basic usage example
│   │   └── README.md
│   └── complete/
│       ├── main.tf        # Full-featured usage example
│       └── README.md
└── tests/
    ├── basic.tftest.hcl   # Module tests
    └── complete.tftest.hcl
```

## File-by-File Breakdown

### `versions.tf` - Constraints

```hcl
# modules/my-module/versions.tf

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

### `variables.tf` - Inputs

```hcl
# modules/my-module/variables.tf

variable "name" {
  type        = string
  description = "(Required) Name of the resource."
}

variable "environment" {
  type        = string
  description = "(Required) Deployment environment: dev, staging, or prod."

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be one of: dev, staging, prod."
  }
}

variable "tags" {
  type        = map(string)
  description = "(Optional) Additional resource tags."
  default     = {}
}
```

### `locals.tf` - Internal Computations

```hcl
# modules/my-module/locals.tf

locals {
  common_tags = merge(
    {
      Name        = var.name
      Environment = var.environment
      ManagedBy   = "OpenTofu"
    },
    var.tags
  )

  name_prefix = lower("${var.name}-${var.environment}")
}
```

### `data.tf` - Data Sources

```hcl
# modules/my-module/data.tf

data "aws_region" "current" {}
data "aws_caller_identity" "current" {}

data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-22.04-amd64-server-*"]
  }
}
```

### `main.tf` - Resources

```hcl
# modules/my-module/main.tf

resource "aws_instance" "this" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  subnet_id     = var.subnet_id

  tags = local.common_tags
}
```

### `outputs.tf` - Outputs

```hcl
# modules/my-module/outputs.tf

output "id" {
  description = "ID of the created resource."
  value       = aws_instance.this.id
}

output "arn" {
  description = "ARN of the created resource."
  value       = aws_instance.this.arn
}
```

## Naming Conventions

- **Files**: lowercase, hyphens (e.g., `security-groups.tf` for complex modules)
- **Resources**: use `this` for the primary resource in a single-resource module
- **Variables**: descriptive, underscore_separated
- **Outputs**: match the attribute name of the primary resource

## README Structure

````markdown
# Module Name

Brief description of what this module does.

## Usage

```hcl
module "example" {
  source = "./modules/my-module"
  name   = "example"
}
```

## Requirements

| Name | Version |
|------|---------|
| opentofu | >= 1.6 |
| aws | >= 5.0 |

## Inputs

| Name | Type | Default | Required |
|------|------|---------|----------|
| name | string | - | yes |

## Outputs

| Name | Description |
|------|-------------|
| id | Resource ID |
````

## Conclusion

A consistent module directory structure makes OpenTofu modules discoverable, understandable, and maintainable. Follow the convention of separating variables, resources, outputs, and data sources into dedicated files, and always include a README with usage examples and an input/output table.
