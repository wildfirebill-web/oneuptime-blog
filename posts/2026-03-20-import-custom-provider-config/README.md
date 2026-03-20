# How to Import Resources with Custom Provider Configurations in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to specify custom provider configurations when importing resources in OpenTofu, enabling cross-account, multi-region, and aliased provider imports.

## Introduction

By default, import blocks use the default provider configuration. However, when importing resources that require a specific provider configuration — such as resources in a different AWS account, a specific region, or requiring an IAM role assumption — you can specify the provider with the `provider` argument.

## Basic Provider Specification

```hcl
# Provider configurations
provider "aws" {
  region = "us-east-1"
  alias  = "us_east"
}

provider "aws" {
  region = "eu-west-1"
  alias  = "eu_west"
}

# Import using the eu-west-1 provider
import {
  provider = aws.eu_west  # Specify which provider configuration to use
  to       = aws_instance.eu_web
  id       = "i-0123456789abcdef0"
}

resource "aws_instance" "eu_web" {
  provider      = aws.eu_west
  ami           = "ami-0european01234"
  instance_type = "t3.micro"
}
```

## Cross-Account Import

```hcl
# Production account provider (role assumption)
provider "aws" {
  region = "us-east-1"
  alias  = "production"

  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformAdmin"
  }
}

# Import resources from the production account
import {
  provider = aws.production
  to       = aws_vpc.prod_main
  id       = "vpc-0a1b2c3d4e5f6789"
}

resource "aws_vpc" "prod_main" {
  provider   = aws.production
  cidr_block = "10.0.0.0/16"
}
```

## Multi-Region Imports

```hcl
# Multiple AWS region providers
provider "aws" {
  region = "us-east-1"
  alias  = "primary"
}

provider "aws" {
  region = "us-west-2"
  alias  = "dr"
}

# Import from primary region
import {
  provider = aws.primary
  to       = aws_vpc.primary
  id       = "vpc-east-0a1b2c3d"
}

# Import from DR region
import {
  provider = aws.dr
  to       = aws_vpc.dr
  id       = "vpc-west-0e1f2a3b"
}
```

## Importing with Different Authentication

```hcl
# Provider with explicit credentials
provider "aws" {
  region     = "us-east-1"
  access_key = var.legacy_access_key
  secret_key = var.legacy_secret_key
  alias      = "legacy_account"
}

import {
  provider = aws.legacy_account
  to       = aws_s3_bucket.legacy_data
  id       = "my-legacy-bucket-name"
}
```

## Azure Multi-Subscription Import

```hcl
provider "azurerm" {
  features {}
  subscription_id = "sub-id-1"
  alias           = "subscription_1"
}

provider "azurerm" {
  features {}
  subscription_id = "sub-id-2"
  alias           = "subscription_2"
}

import {
  provider = azurerm.subscription_2
  to       = azurerm_resource_group.app
  id       = "/subscriptions/sub-id-2/resourceGroups/rg-app"
}
```

## GCP Multi-Project Import

```hcl
provider "google" {
  project = "project-prod"
  region  = "us-central1"
  alias   = "production"
}

provider "google" {
  project = "project-dev"
  region  = "us-central1"
  alias   = "development"
}

import {
  provider = google.production
  to       = google_storage_bucket.app
  id       = "prod-app-bucket"
}
```

## CLI Import with Provider Aliases

For CLI imports, the provider is determined by the resource's `provider` argument:

```hcl
resource "aws_instance" "eu_web" {
  provider = aws.eu_west  # This provider is used for CLI import too
}
```

```bash
# CLI import uses the provider configured on the resource
tofu import aws_instance.eu_web i-0123456789abcdef0
```

## Conclusion

Custom provider configurations for imports are essential in multi-account, multi-region, and hybrid cloud environments. Always specify the `provider` argument in import blocks when the resource requires a non-default provider configuration. This ensures the import uses the correct credentials, region, and account context, and that the resource continues to be managed by the correct provider after import.
