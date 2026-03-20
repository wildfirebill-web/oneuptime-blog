# How to Use Early Variable Evaluation Introduced in OpenTofu 1.8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Early Evaluation, Variables, OpenTofu 1.8, Infrastructure as Code

Description: Learn how to use early variable evaluation in OpenTofu 1.8 to reference input variables in provider and backend configurations.

## Introduction

OpenTofu 1.8 introduced early variable evaluation, allowing input variables to be used in provider configuration and backend configuration blocks. Previously, these blocks could only reference environment variables or hardcoded values. This makes configurations more dynamic and eliminates the need for wrapper scripts to inject values.

## Before OpenTofu 1.8

```hcl
# Previously, you couldn't do this:

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

provider "aws" {
  region = var.aws_region  # ERROR in < 1.8: variables not allowed here
}
```

## Using Variables in Provider Configuration (1.8+)

```hcl
variable "aws_region" {
  type        = string
  description = "AWS region to deploy to"
  default     = "us-east-1"
}

variable "aws_assume_role_arn" {
  type        = string
  description = "IAM role ARN to assume"
  default     = ""
}

provider "aws" {
  region = var.aws_region  # Now works in OpenTofu 1.8+

  dynamic "assume_role" {
    for_each = var.aws_assume_role_arn != "" ? [1] : []
    content {
      role_arn = var.aws_assume_role_arn
    }
  }
}
```

## Using Variables in Backend Configuration

```hcl
variable "state_bucket" {
  type        = string
  description = "S3 bucket name for Terraform state"
}

variable "state_key_prefix" {
  type    = string
  default = "terraform"
}

variable "environment" {
  type = string
}

terraform {
  backend "s3" {
    bucket         = var.state_bucket       # Now works in 1.8+
    key            = "${var.state_key_prefix}/${var.environment}/terraform.tfstate"
    region         = var.aws_region
    encrypt        = true
    dynamodb_table = "${var.state_bucket}-locks"
  }
}
```

## Multi-Region Dynamic Providers

Early evaluation enables truly dynamic multi-region configurations.

```hcl
variable "regions" {
  type    = list(string)
  default = ["us-east-1", "eu-west-1", "ap-southeast-1"]
}

# Create a provider per region (requires provider for_each – OpenTofu 1.9)
provider "aws" {
  alias  = "us_east_1"
  region = var.regions[0]
}

provider "aws" {
  alias  = "eu_west_1"
  region = var.regions[1]
}
```

## Backend Config with Variable Defaults

```hcl
# vars.tfvars
state_bucket     = "my-company-terraform-state"
state_key_prefix = "prod"
environment      = "production"
aws_region       = "us-east-1"
```

```bash
# Initialize with variable values
tofu init \
  -var-file="vars.tfvars" \
  -backend=true
```

## Constraints

Early evaluation has some constraints:
- Variables used in provider/backend blocks cannot have complex expressions
- Variables must be provided before init/plan (via `-var`, `-var-file`, or env vars)
- Variables used in backend blocks are evaluated at `init` time

```bash
# Supply variables at init time for backend configuration
tofu init \
  -var="state_bucket=my-state-bucket" \
  -var="environment=prod"
```

## Summary

Early variable evaluation in OpenTofu 1.8 eliminates a major usability gap by allowing input variables in provider and backend configurations. This simplifies multi-environment setups, removes the need for backend configuration partials, and makes configurations more self-contained and dynamic.
