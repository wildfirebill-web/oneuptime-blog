# How to Import Resources with Custom Provider Configurations in OpenTofu (2)

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Import, Provider

Description: Learn how to import existing resources when your configuration uses custom provider configurations, aliases, or non-default provider instances.

## Introduction

When your configuration uses provider aliases (for multi-region or multi-account deployments) or explicitly configured providers, import blocks and the CLI import command need to reference the correct provider instance. This ensures the resource is imported using the right credentials and region rather than the default provider.

## Import with a Provider Alias

```hcl
# Provider with alias for a secondary region

provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu"
  region = "eu-west-1"
}

# Resource using the aliased provider
resource "aws_s3_bucket" "eu_data" {
  provider = aws.eu
  bucket   = "acme-data-eu-west-1"
}

# Import block must specify the provider
import {
  provider = aws.eu  # Use the aliased provider
  to       = aws_s3_bucket.eu_data
  id       = "acme-data-eu-west-1"
}
```

## Import with Multi-Account Provider

```hcl
provider "aws" {
  alias = "prod_account"
  assume_role {
    role_arn = "arn:aws:iam::111111111111:role/TofuRole"
  }
  region = "us-east-1"
}

resource "aws_vpc" "prod" {
  provider   = aws.prod_account
  cidr_block = "10.0.0.0/16"
}

import {
  provider = aws.prod_account
  to       = aws_vpc.prod
  id       = "vpc-0abc123456"
}
```

## Import via CLI with Custom Provider

For the CLI `tofu import` command, the provider is inferred from the resource address - no explicit flag needed. The resource's `provider` meta-argument in its configuration determines which provider is used:

```bash
# OpenTofu reads the provider from the resource configuration
tofu import aws_s3_bucket.eu_data acme-data-eu-west-1
# Uses aws.eu provider because the resource declares: provider = aws.eu
```

## Import into Module with Custom Provider

```hcl
# Module passes a custom provider
module "eu_region" {
  source = "./modules/region"
  providers = {
    aws = aws.eu
  }
}

# Import into the module's resource using the right provider context
import {
  provider = aws.eu
  to       = module.eu_region.aws_s3_bucket.data
  id       = "acme-eu-data-bucket"
}
```

## Multiple Provider Imports

```hcl
# Import resources across multiple provider instances
import {
  provider = aws.us_east
  to       = aws_s3_bucket.us_data
  id       = "acme-us-east-data"
}

import {
  provider = aws.eu_west
  to       = aws_s3_bucket.eu_data
  id       = "acme-eu-west-data"
}

import {
  provider = aws.ap_southeast
  to       = aws_s3_bucket.ap_data
  id       = "acme-ap-data"
}
```

## Google Cloud with Custom Provider

```hcl
provider "google" {
  alias   = "staging"
  project = "acme-staging"
  region  = "us-central1"
}

resource "google_storage_bucket" "staging_state" {
  provider = google.staging
  name     = "acme-staging-state"
  location = "US"
}

import {
  provider = google.staging
  to       = google_storage_bucket.staging_state
  id       = "acme-staging/acme-staging-state"
}
```

## Verifying Correct Provider on Import

```bash
tofu plan

# The plan shows which provider will handle the import
# # aws_s3_bucket.eu_data will be imported
# # (config refers to provider[registry.opentofu.org/hashicorp/aws].eu)
```

## Conclusion

Import blocks and CLI imports respect the `provider` meta-argument on the target resource. For resources using aliased providers (multi-region, multi-account), add `provider = <alias>` to the import block to use the correct provider instance. The correct provider determines which credentials and region are used when querying the resource during import.
