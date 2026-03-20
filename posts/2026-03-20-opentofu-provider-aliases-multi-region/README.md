# How to Use Provider Aliases for Multi-Region Deployments in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider

Description: Learn how to use provider aliases in OpenTofu to configure multiple instances of the same provider for multi-region and multi-account deployments.

## Introduction

Provider aliases allow you to configure multiple instances of the same provider with different settings. This is essential for deploying resources across multiple AWS regions, multiple GCP projects, or multiple cloud accounts from a single OpenTofu configuration.

## Defining Provider Aliases

Add the `alias` argument to create a named provider instance:

```hcl
# Default provider (no alias) - used for most resources

provider "aws" {
  region = "us-east-1"
}

# Aliased provider - used for specific resources
provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_east"
  region = "ap-east-1"
}
```

## Using Aliases on Resources

Reference an alias with the `provider` meta-argument:

```hcl
# Uses the default provider (us-east-1)
resource "aws_s3_bucket" "primary" {
  bucket = "my-app-primary"
}

# Explicitly uses the eu-west provider
resource "aws_s3_bucket" "eu_replica" {
  provider = aws.eu_west
  bucket   = "my-app-eu-replica"
}

# Uses the ap-east provider
resource "aws_s3_bucket" "ap_replica" {
  provider = aws.ap_east
  bucket   = "my-app-ap-replica"
}
```

## Multi-Region VPC Deployment

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

resource "aws_vpc" "us_east" {
  cidr_block = "10.0.0.0/16"
  tags       = { Name = "vpc-us-east", Region = "us-east-1" }
}

resource "aws_vpc" "us_west" {
  provider   = aws.us_west
  cidr_block = "10.1.0.0/16"
  tags       = { Name = "vpc-us-west", Region = "us-west-2" }
}
```

## Multi-Account Deployments with IAM Role Assumption

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "shared_services"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::222222222222:role/TerraformRole"
  }
}

provider "aws" {
  alias  = "security_account"
  region = "us-east-1"

  assume_role {
    role_arn = "arn:aws:iam::333333333333:role/TerraformRole"
  }
}
```

## Passing Aliases to Modules

Use the `providers` argument to pass aliased providers to modules:

```hcl
module "networking_us" {
  source = "./modules/networking"
  providers = {
    aws = aws  # Default provider
  }
}

module "networking_eu" {
  source = "./modules/networking"
  providers = {
    aws = aws.eu_west  # Aliased provider
  }
}
```

## Data Sources with Aliases

```hcl
# Look up AMIs in different regions
data "aws_ami" "us_east" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

data "aws_ami" "eu_west" {
  provider    = aws.eu_west
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}
```

## Conclusion

Provider aliases are the foundation of multi-region and multi-account infrastructure in OpenTofu. Define one provider block per region or account, assign each a unique alias, and use the `provider` argument on resources or the `providers` argument on modules to route each resource to the correct API endpoint. This pattern scales from two regions to dozens of accounts.
