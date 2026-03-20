# How to Upgrade OpenTofu from 1.7 to 1.8

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Upgrade, Version Migration, Infrastructure as Code, DevOps

Description: A step-by-step guide to safely upgrading your OpenTofu installation from version 1.7 to 1.8.

## Introduction

OpenTofu 1.8 introduced early variable/local evaluation, the `tofu.applying` built-in, and other significant improvements. This guide covers the upgrade process from 1.7 to 1.8.

## What's New in OpenTofu 1.8

- **Early variable evaluation**: Use variables in `backend` and `module source` blocks
- **`tofu.applying`**: Differentiate plan vs apply in configurations
- **Provider iteration**: for_each with providers
- **Improved test framework**: Better `tofu test` capabilities

## Pre-Upgrade Checklist

```bash
# Check current version
tofu version
# OpenTofu v1.7.x

# Create state backup
cp terraform.tfstate terraform.tfstate.backup-before-1.8

# Run a plan to document current state
tofu plan -out=pre-upgrade.tfplan

# Commit current state
git add -A && git commit -m "Pre-upgrade snapshot before OpenTofu 1.8"
```

## Installing OpenTofu 1.8

```bash
# Using tofuenv
tofuenv install latest:^1.8
tofuenv use 1.8.0

# Direct binary
TOFU_VERSION="1.8.0"
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"
unzip "tofu_${TOFU_VERSION}_linux_amd64.zip"
sudo mv tofu /usr/local/bin/

# Verify
tofu version
```

## Update Configuration

```hcl
# versions.tf
terraform {
  # Update to allow 1.8.x
  required_version = ">= 1.8.0"
}
```

## Using New 1.8 Features

### Early Variable Evaluation

```hcl
# 1.8+ allows variables in backend configurations
variable "env" {
  type    = string
  default = "dev"
}

terraform {
  required_version = ">= 1.8.0"

  backend "s3" {
    # Now you can use variables here!
    bucket = "my-tofu-state-${var.env}"
    key    = "terraform.tfstate"
    region = "us-east-1"
  }
}
```

### tofu.applying Built-in

```hcl
# Differentiate between plan and apply
resource "null_resource" "example" {
  triggers = {
    # Only trigger during apply, not plan
    is_applying = tofu.applying ? "true" : "false"
  }
}

output "phase" {
  value = tofu.applying ? "Applying!" : "Planning..."
}
```

## Running Post-Upgrade Validation

```bash
# Re-initialize with upgrade flag
tofu init -upgrade

# Validate configuration
tofu validate

# Run plan and compare with pre-upgrade plan
tofu plan

# Check for deprecation warnings in output
```

## Rollback Procedure

```bash
# If rollback needed
tofuenv use 1.7.x
cp terraform.tfstate.backup-before-1.8 terraform.tfstate

# Revert version constraint
git checkout -- versions.tf
```

## Conclusion

The upgrade from OpenTofu 1.7 to 1.8 is smooth with no required configuration changes for most users. The early variable evaluation feature significantly improves the flexibility of backend configurations, while `tofu.applying` enables sophisticated plan/apply differentiation. Always backup state and test in non-production before upgrading production infrastructure.
