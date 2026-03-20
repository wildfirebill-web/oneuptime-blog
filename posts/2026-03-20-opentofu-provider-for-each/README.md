# How to Use Provider for_each for Dynamic Provider Instances in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Providers

Description: Learn how to use provider for_each in OpenTofu to dynamically create multiple provider instances from a map without writing a separate provider block for each.

## Introduction

OpenTofu supports `for_each` on provider blocks, allowing you to create multiple provider instances from a map. This eliminates the need to write one provider block per account, region, or workspace — instead, you define the provider once and let `for_each` generate all instances.

## Syntax

```hcl
provider "PROVIDER" {
  for_each = MAP

  # Use each.key and each.value to configure each instance
  alias  = each.key
}
```

## Multi-Region AWS Example

```hcl
variable "aws_regions" {
  type = map(object({
    cidr = string
  }))
  default = {
    us-east-1 = { cidr = "10.0.0.0/16" }
    eu-west-1 = { cidr = "10.1.0.0/16" }
    ap-east-1 = { cidr = "10.2.0.0/16" }
  }
}

provider "aws" {
  for_each = var.aws_regions
  alias    = each.key
  region   = each.key
}
```

## Using Dynamic Providers with Modules

Pass dynamically created providers to module instances using `for_each`:

```hcl
module "regional_vpc" {
  for_each = var.aws_regions

  source = "./modules/vpc"
  providers = {
    aws = aws[each.key]  # Reference the dynamic provider instance
  }

  name = "vpc-${each.key}"
  cidr = each.value.cidr
}
```

## Multi-Account Deployment

```hcl
variable "aws_accounts" {
  type = map(object({
    role_arn = string
    region   = string
  }))
  default = {
    production = {
      role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
      region   = "us-east-1"
    }
    staging = {
      role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
      region   = "us-east-1"
    }
  }
}

provider "aws" {
  for_each = var.aws_accounts
  alias    = each.key
  region   = each.value.region

  assume_role {
    role_arn = each.value.role_arn
  }
}

module "account_baseline" {
  for_each = var.aws_accounts

  source = "./modules/account-baseline"
  providers = {
    aws = aws[each.key]
  }

  account_name = each.key
}
```

## Provider for_each with Kubernetes

```hcl
variable "eks_clusters" {
  type = map(object({
    endpoint               = string
    cluster_ca_certificate = string
    token                  = string
  }))
}

provider "kubernetes" {
  for_each = var.eks_clusters
  alias    = each.key

  host                   = each.value.endpoint
  cluster_ca_certificate = base64decode(each.value.cluster_ca_certificate)
  token                  = each.value.token
}
```

## Important Notes

- Provider `for_each` is an OpenTofu-specific feature not available in Terraform.
- The `alias` must be set to `each.key` so each instance has a unique identifier.
- Values used in `for_each` must be known before `tofu init` runs.
- Reference dynamic providers using bracket notation: `aws[each.key]`.

## Conclusion

Provider `for_each` eliminates repetitive provider block declarations for multi-region and multi-account configurations. Define the provider topology as a map variable and let OpenTofu generate all instances dynamically. Combined with module `for_each`, this pattern scales cleanly to dozens of regions or accounts without any code duplication.
