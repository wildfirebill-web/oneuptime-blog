# How to Fix 'Error: Variables May Not Be Used Here' in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, Variables, Error, Backend Configuration, Infrastructure as Code

Description: Learn why OpenTofu prohibits input variables in certain contexts like backend blocks and provider meta-arguments, and how to work around these restrictions.

## Introduction

OpenTofu evaluates certain blocks - most notably `backend` and some provider meta-arguments - during initialization, before input variables are available. Using a variable in these positions produces the "variables may not be used here" error.

## Error Message

```text
Error: Variables not allowed
  on backend.tf line 5, in terraform:
  5:     bucket = var.state_bucket
  Variables may not be used here.

Error: References to input variables not supported here
  on main.tf line 3, in terraform:
  3:     required_version = var.tofu_version
```

## Fix 1: Backend Block - Use -backend-config Instead

The backend block is evaluated at `tofu init` time, before variables exist:

```hcl
# WRONG - variables not allowed in backend

terraform {
  backend "s3" {
    bucket = var.state_bucket   # Error!
    key    = "${var.environment}/tofu.tfstate"
  }
}

# CORRECT - use an empty backend block
terraform {
  backend "s3" {}
}
```

Then pass configuration at init time:

```bash
# Option 1: -backend-config flags
tofu init \
  -backend-config="bucket=my-opentofu-state-prod" \
  -backend-config="key=prod/app/tofu.tfstate" \
  -backend-config="region=us-east-1"

# Option 2: -backend-config file
cat > backend-prod.hcl <<EOF
bucket = "my-opentofu-state-prod"
key    = "prod/app/tofu.tfstate"
region = "us-east-1"
EOF

tofu init -backend-config=backend-prod.hcl
```

## Fix 2: required_version - Use a Literal

`required_version` in the `terraform` block must be a literal string:

```hcl
# WRONG
terraform {
  required_version = var.tofu_version   # Not allowed
}

# CORRECT - use a literal version constraint
terraform {
  required_version = ">= 1.8"
}
```

## Fix 3: Provider Source - Use Literals

Provider source addresses and version constraints are static:

```hcl
# WRONG
terraform {
  required_providers {
    aws = {
      source  = var.aws_provider_source   # Not allowed
      version = var.aws_provider_version
    }
  }
}

# CORRECT
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Fix 4: Provider Assume Role - Use Environment Variables

For dynamic provider configuration (e.g., different roles per environment), use environment variables since the provider block is evaluated before full variable resolution:

```bash
# Set dynamically in CI/CD
export AWS_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/deploy-role"

# Or use a wrapper script that sets the env var before calling tofu
```

```hcl
# Read from environment in the provider block using null-safe expressions
provider "aws" {
  region = "us-east-1"
  # Role can be set via AWS_ROLE_ARN environment variable
  # No need to use a variable in the provider block
}
```

## Fix 5: Workspaces for Environment-Specific Config

Use `terraform.workspace` (a special built-in) instead of a variable for environment-specific values:

```hcl
# terraform.workspace IS allowed in some contexts
locals {
  environment = terraform.workspace   # "default", "prod", "staging"
}

terraform {
  backend "s3" {}   # Still can't use locals here - use -backend-config
}
```

## Conclusion

Variables are prohibited in `backend`, `required_version`, `required_providers`, and provider source/version fields because these are evaluated during initialization before the variable evaluation phase. Solve this by passing backend config via `-backend-config`, using literal values for version constraints, and using environment variables for dynamic provider settings.
