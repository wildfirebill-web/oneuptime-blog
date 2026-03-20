# How to Publish a Module to the OpenTofu Registry

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Modules, Registry

Description: Learn how to publish your OpenTofu modules to the public registry so the community can discover and use them.

Publishing a module to the OpenTofu Registry (or Terraform Registry) allows others to discover and use your infrastructure code. The registry provides versioning, documentation, and a trusted distribution channel for reusable modules.

## Prerequisites

Before publishing, your module must meet these requirements:

1. **GitHub repository** - The module must be on GitHub (public)
2. **Naming convention** - Repository name must follow `terraform-<PROVIDER>-<NAME>`
3. **Standard structure** - Follow the standard module file layout
4. **Semantic versioning** - Use Git tags like `v1.0.0`
5. **README.md** - Documentation at the root level

## Repository Naming Convention

```text
terraform-aws-vpc
terraform-google-kubernetes
terraform-azurerm-database
terraform-helm-redis
```

The format is: `terraform-{provider}-{module-name}`

## Required File Structure

```text
terraform-aws-webapp/
├── main.tf          # Main resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── versions.tf      # Provider requirements
├── README.md        # Module documentation
└── examples/
    └── basic/
        ├── main.tf
        └── README.md
```

## Writing a Publishable Module

```hcl
# versions.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0"
    }
  }
}

# variables.tf
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "tags" {
  description = "Additional tags to apply to resources"
  type        = map(string)
  default     = {}
}

# outputs.tf
output "vpc_id" {
  description = "ID of the created VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}
```

## Creating a GitHub Release (Git Tag)

```bash
# Tag your release
git tag v1.0.0
git push origin v1.0.0

# Or create annotated tag
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0
```

## Registering on the OpenTofu Registry

1. Sign in to `registry.opentofu.org` with GitHub
2. Click "Publish" > "Module"
3. Select your repository
4. Confirm the module details
5. Click "Publish Module"

## Versioning Your Module

Follow semantic versioning:

```bash
# Patch: bug fixes (1.0.0 -> 1.0.1)
git tag v1.0.1

# Minor: new features, backward compatible (1.0.1 -> 1.1.0)
git tag v1.1.0

# Major: breaking changes (1.1.0 -> 2.0.0)
git tag v2.0.0

git push origin --tags
```

## Using a Published Module

```hcl
module "vpc" {
  source  = "registry.opentofu.org/myorg/vpc/aws"
  version = "~> 1.2"

  vpc_cidr    = "10.0.0.0/16"
  environment = "prod"
  tags = {
    Team    = "platform"
    Project = "infrastructure"
  }
}
```

## Module Documentation Best Practices

````markdown
# terraform-aws-vpc

Creates a VPC with public and private subnets.

## Usage

```hcl
module "vpc" {
  source  = "registry.opentofu.org/myorg/vpc/aws"
  version = "~> 1.0"

  vpc_cidr    = "10.0.0.0/16"
  environment = "prod"
}
```

## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.0 |
| aws | >= 4.0 |

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| vpc_cidr | CIDR block | string | "10.0.0.0/16" | no |
| environment | Environment | string | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC identifier |
````

## Conclusion

Publishing modules to the registry increases reusability and helps the community build on proven infrastructure patterns. Follow the naming conventions, maintain good documentation, and use semantic versioning to make your modules production-ready.
