# How to Pass Providers Between Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider, Module

Description: Learn how to pass provider configurations between modules in OpenTofu using the providers argument and proxy provider declarations.

## Introduction

When a root module has multiple provider aliases or custom provider configurations, child modules need explicit instructions about which provider instance to use. The `providers` argument on a module block maps root-module provider addresses to the provider names expected inside the child module.

## Default Inheritance

Without explicit passing, child modules inherit the default (non-aliased) provider:

```hcl
# Root module

provider "aws" {
  region = "us-east-1"
}

# Child module inherits the default aws provider automatically
module "vpc" {
  source = "./modules/vpc"
}
```

## Passing an Aliased Provider

```hcl
# Root module
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "dr"
  region = "us-west-2"
}

# Pass the DR provider to the disaster recovery module
module "dr_infrastructure" {
  source = "./modules/vpc"
  providers = {
    aws = aws.dr  # Map the module's "aws" to the root's "aws.dr"
  }

  name = "disaster-recovery"
}
```

## Child Module: Declaring Expected Providers

Child modules declare their provider requirements in `required_providers`:

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

The child module uses `aws` as its local name - the root module maps whichever alias to fill that slot.

## Passing Multiple Providers

```hcl
# Root module
provider "aws" {
  region = "us-east-1"
}

provider "cloudflare" {
  api_token = var.cloudflare_token
}

module "application" {
  source = "./modules/app"
  providers = {
    aws        = aws
    cloudflare = cloudflare
  }
}
```

## Proxy Provider Declarations (for Module Authors)

When writing a module that will pass a provider to a nested sub-module, declare the intermediate provider as a proxy:

```hcl
# modules/app/versions.tf - intermediate module
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.0"
      configuration_aliases = [aws.alternate]  # Accepts an alternate alias
    }
  }
}
```

```hcl
# modules/app/main.tf - pass the alternate provider to a nested module
module "replica" {
  source = "./replica"
  providers = {
    aws = aws.alternate
  }
}
```

## Explicit vs Implicit Passing

```hcl
# Implicit (works for default providers)
module "vpc" {
  source = "./modules/vpc"
  # aws provider is inherited automatically
}

# Explicit (required when using aliases or non-default configs)
module "vpc_eu" {
  source = "./modules/vpc"
  providers = {
    aws = aws.eu_west  # Must be explicit for aliases
  }
}
```

## Conclusion

Provider passing is how multi-region and multi-account configurations flow through the module tree. The `providers` argument on a module block maps root-level provider aliases to the names the child module expects. Child modules should declare `required_providers` to document which providers they need, and intermediate modules should use `configuration_aliases` when they forward providers to nested sub-modules.
