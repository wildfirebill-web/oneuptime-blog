# How to Fix "Error: Invalid Provider Configuration" in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Provider Configuration, Error, Infrastructure as Code, Debugging

Description: Learn how to diagnose and fix invalid provider configuration errors in OpenTofu, covering missing required attributes, incorrect types, and provider alias issues.

## Introduction

"Error: Invalid provider configuration" surfaces when the provider block contains wrong attribute types, missing required fields, or references to undefined variables. It can also appear when using `provider` meta-arguments that reference an alias that does not exist.

## Common Error Forms

```
Error: Invalid provider configuration
  Provider "registry.opentofu.org/hashicorp/aws" requires explicit configuration.
  Add a provider block to the root module with necessary arguments, or pass
  the provider configuration to this module using the "providers" argument.

Error: No value for required variable
  The root module input variable "aws_region" is not set, and has no default value.
  Use a -var or -var-file command line argument to provide a value for this variable.

Error: Unsupported argument
  An argument named "assume_role" is not expected here. Did you mean "assume_role_arn"?
```

## Fix 1: Add Required Provider Arguments

Check the provider's documentation for required arguments:

```hcl
# WRONG — missing required region
provider "aws" {}

# CORRECT
provider "aws" {
  region = "us-east-1"
}

# Or via environment variable (no explicit region needed)
# export AWS_DEFAULT_REGION=us-east-1
```

## Fix 2: Fix Incorrect Attribute Names

```hcl
# WRONG — attribute name typo
provider "aws" {
  region    = "us-east-1"
  assume_role_arn = "arn:aws:iam::123456789012:role/role-name"  # Wrong
}

# CORRECT
provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::123456789012:role/role-name"
  }
}
```

## Fix 3: Provider Alias Mismatch

```hcl
# WRONG — resource references undefined alias
resource "aws_instance" "west" {
  provider = aws.us-west-2   # This alias doesn't exist
  ami      = "ami-abc123"
  instance_type = "t3.micro"
}

# CORRECT — define the alias first
provider "aws" {
  alias  = "us-west-2"
  region = "us-west-2"
}

resource "aws_instance" "west" {
  provider = aws.us-west-2
  ami      = "ami-abc123"
  instance_type = "t3.micro"
}
```

## Fix 4: Passing Provider to a Module

If a module uses a non-default provider, you must pass it explicitly:

```hcl
# WRONG — module uses aws.west but it's not passed
module "west_vpc" {
  source = "./modules/vpc"
}

# CORRECT
module "west_vpc" {
  source = "./modules/vpc"
  providers = {
    aws = aws.us-west-2
  }
}
```

## Fix 5: Variable Used in Provider Config Not Set

Provider configurations are evaluated before variables are fully resolved. Use `TF_VAR_` or `-var` flags:

```hcl
variable "aws_region" {
  type = string
}

provider "aws" {
  region = var.aws_region
}
```

```bash
# Provide the value
tofu plan -var="aws_region=us-east-1"
# Or
export TF_VAR_aws_region=us-east-1
tofu plan
```

## Validating Provider Configuration

```bash
# Run validate to catch configuration errors before plan
tofu validate

# Check which providers are configured
tofu providers
```

## Conclusion

Invalid provider configuration errors usually come down to: missing required attributes, attribute name typos, undefined provider aliases, or missing variable values. Use `tofu validate` to catch these early, check provider documentation for exact attribute names, and ensure all aliases referenced by resources and modules are defined.
