# How to Manage Multiple AWS Accounts in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, AWS, Terraform, IaC, DevOps, Multi-Account

Description: Learn how to manage infrastructure across multiple AWS accounts in OpenTofu using provider aliases, assume_role, and cross-account resource patterns.

## Introduction

Organizations using AWS Organizations typically split workloads across multiple accounts for isolation, billing, and governance. OpenTofu supports multi-account deployments using provider aliases combined with `assume_role`. Each alias represents a provider that assumes a role in a different target account.

## Multi-Account Provider Pattern

```hcl
# CI/CD account: where OpenTofu runs

# Has credentials that can assume roles in other accounts

# Management/shared services account
provider "aws" {
  alias  = "shared"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.shared_account_id}:role/opentofu-deploy"
  }
}

# Production account
provider "aws" {
  alias  = "production"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.prod_account_id}:role/opentofu-deploy"
  }
}

# Staging account
provider "aws" {
  alias  = "staging"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.staging_account_id}:role/opentofu-deploy"
  }
}

# Development account
provider "aws" {
  alias  = "development"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::${var.dev_account_id}:role/opentofu-deploy"
  }
}
```

## Deploying Resources to Specific Accounts

```hcl
# Route53 hosted zone in shared account (DNS is centralized)
resource "aws_route53_zone" "main" {
  provider = aws.shared
  name     = var.domain_name
}

# Production application resources
resource "aws_instance" "prod_app" {
  provider      = aws.production
  ami           = var.ami_id
  instance_type = "m5.large"
}

# Staging application resources
resource "aws_instance" "staging_app" {
  provider      = aws.staging
  ami           = var.ami_id
  instance_type = "t3.medium"
}
```

## Cross-Account Data Sources

```hcl
# Get current account ID
data "aws_caller_identity" "prod" {
  provider = aws.production
}

data "aws_caller_identity" "staging" {
  provider = aws.staging
}

# Look up resources in other accounts
data "aws_vpc" "shared" {
  provider = aws.shared
  id       = var.shared_vpc_id
}
```

## Account Variables Pattern

Centralize account IDs in variables:

```hcl
# variables.tf
variable "aws_accounts" {
  type = object({
    shared     = string
    production = string
    staging    = string
    dev        = string
  })
  description = "AWS account IDs"
}

# terraform.tfvars
aws_accounts = {
  shared     = "111111111111"
  production = "222222222222"
  staging    = "333333333333"
  dev        = "444444444444"
}
```

```hcl
# providers.tf - use account IDs from variable
provider "aws" {
  alias  = "prod"
  region = var.region

  assume_role {
    role_arn = "arn:aws:iam::${var.aws_accounts.production}:role/opentofu-deploy"
  }
}
```

## Cross-Account S3 Access

```hcl
# State bucket in shared account
resource "aws_s3_bucket" "state" {
  provider = aws.shared
  bucket   = "org-opentofu-state"
}

# Allow other accounts to access state
resource "aws_s3_bucket_policy" "state" {
  provider = aws.shared
  bucket   = aws_s3_bucket.state.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        AWS = [
          "arn:aws:iam::${var.aws_accounts.production}:role/opentofu-deploy",
          "arn:aws:iam::${var.aws_accounts.staging}:role/opentofu-deploy"
        ]
      }
      Action   = ["s3:GetObject", "s3:PutObject", "s3:ListBucket"]
      Resource = [
        aws_s3_bucket.state.arn,
        "${aws_s3_bucket.state.arn}/*"
      ]
    }]
  })
}
```

## Module Pattern for Multi-Account

Pass different providers to the same module for each account:

```hcl
module "baseline_production" {
  source = "./modules/account-baseline"

  account_name = "production"
  environment  = "production"

  providers = {
    aws = aws.production
  }
}

module "baseline_staging" {
  source = "./modules/account-baseline"

  account_name = "staging"
  environment  = "staging"

  providers = {
    aws = aws.staging
  }
}
```

## Output Account IDs for Verification

```hcl
data "aws_caller_identity" "current" {}

output "account_ids" {
  value = {
    current    = data.aws_caller_identity.current.account_id
    # Uncomment to verify assume_role is working
    # production = data.aws_caller_identity.prod.account_id
    # staging    = data.aws_caller_identity.staging.account_id
  }
}
```

## Conclusion

Multi-account management in OpenTofu combines provider aliases with `assume_role` to deploy resources to different AWS accounts from a single configuration. Centralize account IDs in variables for maintainability. Use a shared services account for centralized resources like DNS and state storage. Pass appropriate provider aliases to reusable modules via the `providers` argument to make modules account-agnostic. This pattern scales well from small organizations with two accounts to large enterprises with hundreds.
