# How to Pass Providers to Child Modules in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Module, Provider

Description: Learn how to pass provider configurations to child modules in OpenTofu using the providers argument and required_providers declarations.

## Introduction

By default, child modules inherit the default provider configuration from their parent. When you have multiple provider aliases - for example, deploying to several AWS regions - you need to explicitly pass the right provider configuration to each module using the `providers` argument.

## Default Provider Inheritance

If your module only uses one provider with no aliases, it inherits automatically:

```hcl
# Root module

provider "aws" {
  region = "us-east-1"
}

# Child module inherits the default aws provider
module "vpc" {
  source = "./modules/vpc"
  name   = "main"
}
```

## Passing a Provider Alias

When you have provider aliases, use the `providers` map to explicitly assign them:

```hcl
# Root module
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

# Pass the us-east provider to one module
module "vpc_us" {
  source = "./modules/vpc"
  providers = {
    aws = aws.us_east
  }
}

# Pass the eu-west provider to another module
module "vpc_eu" {
  source = "./modules/vpc"
  providers = {
    aws = aws.eu_west
  }
}
```

## Declaring required_providers in Child Modules

Child modules should declare expected providers using `required_providers`:

```hcl
# modules/vpc/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

## Multi-Region Deployment Pattern

```hcl
# providers.tf (root module)
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# main.tf
module "primary_region" {
  source = "./modules/infrastructure"
  providers = {
    aws = aws.primary
  }
  environment = "production"
}

module "dr_region" {
  source = "./modules/infrastructure"
  providers = {
    aws = aws.dr
  }
  environment = "production-dr"
}
```

## Passing Multiple Providers

A module can accept multiple providers:

```hcl
# Root module
provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}

module "web_app" {
  source = "./modules/web-app"
  providers = {
    aws        = aws
    cloudflare = cloudflare
  }
}
```

```hcl
# modules/web-app/versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
    cloudflare = {
      source  = "cloudflare/cloudflare"
      version = ">= 4.0"
    }
  }
}
```

## Important Notes

- If a module uses provider aliases internally, it must re-export them in its own `providers` argument when calling sub-modules.
- Do not configure providers inside child modules - always pass them from the root.
- The `providers` map keys must match the provider local names used inside the module.
- Omitting `providers` causes the module to inherit all default (non-aliased) provider configurations.

## Conclusion

Explicitly passing providers to child modules is essential for multi-region, multi-account, and multi-cloud deployments. Use the `providers` argument to map root-module provider aliases to the names expected inside each child module, and always declare `required_providers` in child modules for clarity and version enforcement.
