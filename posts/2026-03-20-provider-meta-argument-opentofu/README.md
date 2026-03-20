# How to Use the provider Meta-Argument in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resources, Provider, Multi-Region, Infrastructure as Code, DevOps

Description: A guide to using the provider meta-argument in OpenTofu resources to specify which provider configuration to use for multi-region and multi-account deployments.

## Introduction

The `provider` meta-argument lets you specify which provider configuration to use for a resource when multiple configurations of the same provider exist. This is essential for multi-region deployments, multi-account AWS setups, and any scenario requiring different provider configurations for different resources.

## Provider Aliases

```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Default provider (no alias)
provider "aws" {
  region = "us-east-1"
}

# Provider with alias for a different region
provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

# Another provider for a different account
provider "aws" {
  alias  = "prod_account"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::PROD_ACCOUNT_ID:role/TerraformRole"
  }
}
```

## Using the provider Meta-Argument

```hcl
# Default provider (us-east-1) - no provider argument needed
resource "aws_vpc" "east" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "east-vpc", Region = "us-east-1" }
}

# Use the us_west alias
resource "aws_vpc" "west" {
  provider   = aws.us_west  # Specify the alias
  cidr_block = "10.1.0.0/16"
  tags = { Name = "west-vpc", Region = "us-west-2" }
}

# Use the prod_account alias
resource "aws_s3_bucket" "prod_state" {
  provider = aws.prod_account
  bucket   = "prod-terraform-state"
}
```

## Multi-Region Infrastructure

```hcl
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "eu_west"
  region = "eu-west-1"
}

provider "aws" {
  alias  = "ap_southeast"
  region = "ap-southeast-1"
}

# Create EC2 instances in multiple regions
resource "aws_instance" "us_east" {
  ami           = "ami-0c55b159cbfafe1f0"  # US East AMI
  instance_type = "t3.micro"
  # Uses default provider (us-east-1)
}

resource "aws_instance" "eu_west" {
  provider      = aws.eu_west
  ami           = "ami-0d70546e43a941d70"  # EU West AMI
  instance_type = "t3.micro"
}

resource "aws_instance" "ap_southeast" {
  provider      = aws.ap_southeast
  ami           = "ami-0801d4ffb40b33b4d"  # AP Southeast AMI
  instance_type = "t3.micro"
}
```

## Multi-Region Route53 with provider

```hcl
provider "aws" {
  region = var.primary_region
}

# Route53 is global, but must use us-east-1 for ACM certs
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

# ACM certificates for CloudFront must be in us-east-1
resource "aws_acm_certificate" "cdn" {
  provider          = aws.us_east_1  # Required for CloudFront
  domain_name       = "*.example.com"
  validation_method = "DNS"
}
```

## Provider in Modules

```hcl
# Passing provider to a module
module "east_vpc" {
  source = "./modules/vpc"

  providers = {
    aws = aws  # Default provider
  }

  cidr_block = "10.0.0.0/16"
}

module "west_vpc" {
  source = "./modules/vpc"

  providers = {
    aws = aws.us_west  # Pass the aliased provider
  }

  cidr_block = "10.1.0.0/16"
}
```

## Conclusion

The `provider` meta-argument is essential for multi-region and multi-account OpenTofu deployments. By creating aliased provider configurations and using the `provider` meta-argument in resource blocks, you can manage infrastructure across different regions and accounts within a single OpenTofu configuration. This is particularly powerful for disaster recovery setups, global applications, and multi-account governance patterns.
