# Using for_each with Providers in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Provider, for_each

Description: Learn how OpenTofu 1.7+ supports for_each on provider blocks, enabling dynamic multi-region and multi-account deployments.

OpenTofu 1.7 introduced the ability to use `for_each` with provider configurations, enabling truly dynamic multi-region and multi-account deployments without repeating provider blocks.

## The Problem Before for_each

```hcl
# Old approach - manual repetition for each region

provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west_2"
  region = "us-west-2"
}

provider "aws" {
  alias  = "eu_west_1"
  region = "eu-west-1"
}

# And then manual resource definitions for each...
resource "aws_s3_bucket" "logs_us_east" {
  provider = aws.us_east_1
  bucket   = "logs-us-east-1"
}
# Repeat for each region...
```

## for_each on Provider Blocks (OpenTofu 1.7+)

```hcl
# Dynamically create providers for each region
variable "regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
}

provider "aws" {
  for_each = var.regions
  alias    = each.key
  region   = each.key
}

# Resources can now reference these dynamic providers
resource "aws_s3_bucket" "regional_logs" {
  for_each = var.regions
  provider = aws[each.key]
  bucket   = "app-logs-${each.key}"
}
```

## Multi-Account Provider Configuration

```hcl
variable "accounts" {
  type = map(object({
    role_arn = string
    region   = string
  }))
  default = {
    dev = {
      role_arn = "arn:aws:iam::111111111111:role/TerraformRole"
      region   = "us-east-1"
    }
    staging = {
      role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
      region   = "us-east-1"
    }
    prod = {
      role_arn = "arn:aws:iam::333333333333:role/TerraformRole"
      region   = "us-east-1"
    }
  }
}

provider "aws" {
  for_each = var.accounts
  alias    = each.key
  region   = each.value.region

  assume_role {
    role_arn = each.value.role_arn
  }
}
```

## Passing for_each Providers to Modules

```hcl
variable "regions" {
  type    = set(string)
  default = ["us-east-1", "eu-west-1"]
}

provider "aws" {
  for_each = var.regions
  alias    = each.key
  region   = each.key
}

module "regional_infrastructure" {
  for_each = var.regions

  source = "./modules/regional"

  providers = {
    aws = aws[each.key]
  }

  region      = each.key
  environment = var.environment
}
```

```hcl
# modules/regional/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}

# modules/regional/main.tf
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = {
    Region = var.region
  }
}

resource "aws_s3_bucket" "logs" {
  bucket = "app-logs-${var.region}-${var.environment}"
}
```

## Global and Regional Resource Pattern

```hcl
# Some resources are global (IAM, Route53) while others are regional
variable "deployment_regions" {
  type    = set(string)
  default = ["us-east-1", "eu-west-1", "ap-northeast-1"]
}

# Default provider for global resources
provider "aws" {
  region = "us-east-1"
}

# Regional providers
provider "aws" {
  for_each = var.deployment_regions
  alias    = each.key
  region   = each.key
}

# Global IAM role (uses default provider)
resource "aws_iam_role" "app" {
  name = "application-role"
  # ...
}

# Regional resources
resource "aws_lambda_function" "app" {
  for_each = var.deployment_regions
  provider = aws[each.key]

  function_name = "app-${each.key}"
  role          = aws_iam_role.app.arn
  # ...
}
```

## Outputs from for_each Providers

```hcl
output "regional_vpc_ids" {
  value = {
    for region in var.regions :
    region => module.regional_infrastructure[region].vpc_id
  }
}
```

## Conclusion

Provider `for_each` in OpenTofu 1.7+ eliminates the boilerplate of repeating provider blocks for multi-region and multi-account setups. It makes your configurations more concise, easier to maintain, and allows regions/accounts to be driven by variables rather than hardcoded. This is particularly valuable for organizations managing infrastructure across many regions or accounts.
