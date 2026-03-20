# How to Configure Terragrunt to Use OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terragrunt, Configuration, Infrastructure as Code, DevOps

Description: Learn how to configure Terragrunt to invoke OpenTofu instead of Terraform, enabling DRY infrastructure management with the open-source OpenTofu toolchain.

## Introduction

Terragrunt is a thin wrapper around OpenTofu (and Terraform) that provides extra tools for keeping your configurations DRY. Since OpenTofu is a drop-in replacement for Terraform, Terragrunt works with it by simply pointing it to the `tofu` binary.

## Installing Prerequisites

```bash
# Install OpenTofu

brew install opentofu    # macOS
# or follow https://opentofu.org/docs/intro/install/

# Install Terragrunt
brew install terragrunt  # macOS
# or download from https://terragrunt.gruntwork.io/

# Verify both are installed
tofu version
terragrunt --version
```

## Configuring Terragrunt to Use OpenTofu

Terragrunt respects the `TERRAGRUNT_TFPATH` environment variable to override which binary it calls:

```bash
# Use tofu for all terragrunt commands in this shell session
export TERRAGRUNT_TFPATH=$(which tofu)

# Verify
terragrunt --version
# Should show OpenTofu version in the output
```

## Permanent Configuration via terragrunt.hcl

You can hard-code the OpenTofu binary path in your root `terragrunt.hcl`:

```hcl
# root terragrunt.hcl
# Tell Terragrunt to use OpenTofu instead of Terraform
terraform_binary = "tofu"

# Remote state backend configuration shared across all modules
remote_state {
  backend = "s3"
  config = {
    bucket         = "my-opentofu-state"
    key            = "${path_relative_to_include()}/tofu.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "opentofu-state-locks"
  }
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
}

# Provider configuration injected into all modules
generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"
  contents  = <<EOF
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
EOF
}
```

## Project Layout

A typical Terragrunt project using OpenTofu looks like this:

```text
infrastructure/
├── terragrunt.hcl          # Root config (sets terraform_binary = "tofu")
├── common_vars.yaml        # Shared variables
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/
│       ├── main.tf
│       └── ...
└── environments/
    ├── dev/
    │   ├── terragrunt.hcl  # Inherits root config
    │   └── vpc/
    │       └── terragrunt.hcl
    └── prod/
        ├── terragrunt.hcl
        └── vpc/
            └── terragrunt.hcl
```

## Child terragrunt.hcl Example

```hcl
# environments/dev/vpc/terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

terraform {
  source = "../../../modules/vpc"
}

inputs = {
  vpc_cidr    = "10.0.0.0/16"
  aws_region  = "us-east-1"
  environment = "dev"
}
```

## Running Terragrunt Commands

```bash
# Plan a single module
terragrunt plan

# Apply a single module
terragrunt apply

# Run across all modules in the environment
terragrunt run-all plan

# Apply all modules respecting dependency order
terragrunt run-all apply
```

## CI/CD Integration

```yaml
# .github/workflows/infra.yml
- name: Install OpenTofu
  run: |
    curl -LO https://github.com/opentofu/opentofu/releases/download/v1.9.0/tofu_1.9.0_linux_amd64.zip
    unzip tofu_1.9.0_linux_amd64.zip && sudo mv tofu /usr/local/bin/

- name: Terragrunt Plan
  env:
    TERRAGRUNT_TFPATH: /usr/local/bin/tofu
  run: terragrunt run-all plan --terragrunt-non-interactive
```

## Conclusion

Configuring Terragrunt to use OpenTofu is straightforward - set `terraform_binary = "tofu"` in your root `terragrunt.hcl` or export `TERRAGRUNT_TFPATH`. From there, all Terragrunt features like DRY backends, dependency management, and `run-all` work seamlessly with OpenTofu.
