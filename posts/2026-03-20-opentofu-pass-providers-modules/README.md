# How to Pass Providers to Child Modules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to pass provider configurations to child modules in OpenTofu for multi-region, multi-account, and multi-cloud deployments.

## Introduction

By default, child modules inherit the provider configurations from their parent (root) module. However, when you need a module to use a specific provider alias - for multi-region or multi-account deployments - you must explicitly pass the provider using the `providers` argument.

## Default Provider Inheritance

```hcl
# Root module - default provider

provider "aws" {
  region = "us-east-1"
}

# Child module inherits the default provider automatically
module "vpc" {
  source = "./modules/vpc"
  # No providers argument needed - inherits default aws provider
}
```

## Passing a Provider Alias

### Step 1: Define Provider Aliases in Root

```hcl
# Root: providers.tf

provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}
```

### Step 2: Declare the Provider in the Module

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

# modules/vpc/main.tf - declare provider in resource
resource "aws_vpc" "this" {
  provider   = aws  # uses the passed-in provider
  cidr_block = var.cidr_block
}
```

### Step 3: Pass Providers to the Module

```hcl
# Root: main.tf

module "vpc_us_east" {
  source = "./modules/vpc"

  providers = {
    aws = aws.us_east
  }

  cidr_block = "10.0.0.0/16"
}

module "vpc_eu_west" {
  source = "./modules/vpc"

  providers = {
    aws = aws.eu_west
  }

  cidr_block = "10.1.0.0/16"
}
```

## Multi-Account Deployment

```hcl
# Root: providers.tf

provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "staging_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::STAGING_ACCOUNT_ID:role/TerraformRole"
  }
}

# Deploy the same module to different accounts
module "prod_infra" {
  source = "./modules/application"

  providers = {
    aws = aws.prod_account
  }

  environment = "prod"
}

module "staging_infra" {
  source = "./modules/application"

  providers = {
    aws = aws.staging_account
  }

  environment = "staging"
}
```

## Multi-Cloud Module

```hcl
provider "aws" {}
provider "google" {
  project = var.gcp_project
}

module "hybrid_network" {
  source = "./modules/hybrid-network"

  providers = {
    aws    = aws
    google = google
  }

  aws_cidr = "10.0.0.0/16"
  gcp_cidr = "10.1.0.0/16"
}
```

## Module with Multiple Provider Requirements

```hcl
# modules/multi-region/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 5.0"
      configuration_aliases = [aws.primary, aws.secondary]
    }
  }
}
```

```hcl
# Root: main.tf
module "multi_region" {
  source = "./modules/multi-region"

  providers = {
    aws.primary   = aws.us_east
    aws.secondary = aws.eu_west
  }
}
```

## Step-by-Step Usage

1. Define provider aliases in root with `alias = "name"`.
2. Pass them to modules using `providers = { aws = aws.aliasname }`.
3. The module uses the passed provider for all resources.

## Conclusion

Passing providers to child modules enables multi-region, multi-account, and multi-cloud deployments from a single module definition. Use the `providers` meta-argument to explicitly route provider configurations to child modules, enabling the same module code to deploy infrastructure in different regions or accounts.
