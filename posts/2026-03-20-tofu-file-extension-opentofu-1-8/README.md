# How to Use the .tofu File Extension Introduced in OpenTofu 1.8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, .tofu Extension, OpenTofu 1.8, Configuration, Infrastructure as Code

Description: Learn how to use the .tofu file extension introduced in OpenTofu 1.8 to write OpenTofu-specific configurations that are ignored by Terraform.

## Introduction

OpenTofu 1.8 introduced support for `.tofu` files as an alternative to `.tf` files. Files with the `.tofu` extension are processed by OpenTofu but ignored by Terraform. This allows you to add OpenTofu-specific features (like state encryption or provider for_each) to a codebase that also needs to remain Terraform-compatible.

## File Extension Precedence

OpenTofu 1.8+ processes files in this order:
1. `.tofu` files take precedence over `.tf` files with the same basename
2. If `main.tofu` and `main.tf` both exist, only `main.tofu` is used
3. `.tf` files without a corresponding `.tofu` file are processed normally

## Using .tofu Files for OpenTofu-Specific Features

```
project/
├── main.tf              # shared config (works in both Terraform and OpenTofu)
├── variables.tf         # shared variables
├── outputs.tf           # shared outputs
├── encryption.tofu      # OpenTofu-specific: state encryption
└── provider-config.tofu # OpenTofu-specific: provider for_each (1.9)
```

```hcl
# encryption.tofu – only processed by OpenTofu
# Terraform will ignore this file

terraform {
  encryption {
    key_provider "aws_kms" "main" {
      kms_key_id = var.state_kms_key_arn
      region     = var.aws_region
    }

    method "aes_gcm" "main" {
      keys = key_provider.aws_kms.main
    }

    state {
      method   = method.aes_gcm.main
      enforced = true
    }
  }
}
```

## Overriding Terraform Configuration

If you have a `.tf` file with content that needs to differ between Terraform and OpenTofu, create a `.tofu` file with the same base name:

```hcl
# provider.tf – works with Terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

```hcl
# provider.tofu – overrides provider.tf for OpenTofu users
# OpenTofu uses this file; Terraform uses provider.tf

terraform {
  required_version = ">= 1.8.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region  # early variable evaluation (OpenTofu 1.8 feature)
}
```

## Variable Override Files

```hcl
# defaults.tf – shared defaults
variable "environment" {
  type    = string
  default = "dev"
}
```

```hcl
# defaults.tofu – OpenTofu-specific validation
variable "environment" {
  type    = string
  default = "dev"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

## Detecting File Type in CI

```bash
# In CI/CD, you can detect whether to use tofu or terraform
if command -v tofu &>/dev/null; then
  TF_CMD="tofu"
else
  TF_CMD="terraform"
fi

$TF_CMD init
$TF_CMD plan
```

## File Extension Summary

| Extension | Processed by | Use for |
|-----------|-------------|---------|
| `.tf`     | Both Terraform and OpenTofu | Shared configuration |
| `.tofu`   | OpenTofu only | OpenTofu-specific features |
| `.tf.json`| Both | JSON-format configuration |
| `.tofu.json` | OpenTofu only | JSON-format OpenTofu-specific config |
| `.tfvars` | Both | Variable values |
| `.tofu.tfvars` | OpenTofu only | OpenTofu-specific variable values |

## Summary

The `.tofu` file extension enables gradual adoption of OpenTofu-specific features in codebases that must remain Terraform-compatible. Use `.tofu` files for encryption configuration, early variable evaluation, provider for_each, and other OpenTofu innovations without breaking Terraform users of the same codebase.
