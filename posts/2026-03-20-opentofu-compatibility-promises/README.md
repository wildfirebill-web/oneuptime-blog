# How to Understand OpenTofu Compatibility Promises

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Compatibility, Versioning, Stability, Best Practices, Infrastructure as Code

Description: Learn about OpenTofu's compatibility promises for language features, state formats, provider protocols, and APIs to plan version upgrades confidently.

## Introduction

Understanding what OpenTofu promises to keep stable across versions helps you make informed decisions about when and how to upgrade. OpenTofu follows semantic versioning and makes specific commitments about backward compatibility for different parts of the system.

## Semantic Versioning

OpenTofu follows `MAJOR.MINOR.PATCH` versioning:

| Version Change | Meaning |
|----------------|---------|
| PATCH (1.9.0 → 1.9.1) | Bug fixes only, always safe to upgrade |
| MINOR (1.9.0 → 1.10.0) | New features, backward compatible |
| MAJOR (1.x → 2.0) | Breaking changes allowed |

## What Is Stable

### Configuration Language (HCL)
Within a major version:
- Existing valid configurations will continue to be valid
- Existing functions will continue to work as documented
- Resource type and argument names are stable

```hcl
# This configuration written for OpenTofu 1.x will continue to work
# across all 1.x patch and minor versions
resource "aws_s3_bucket" "example" {
  bucket = var.bucket_name

  tags = {
    for k, v in var.tags : k => v
  }
}
```

### State Format
- State files from older minor versions can be read by newer minor versions
- State migrations happen automatically on first use
- State format changes are backward compatible within a major version

```bash
# After upgrading OpenTofu, state is automatically migrated
# The old state file format is still readable
tofu state list  # works even after upgrades
```

### Provider Protocol
- OpenTofu commits to supporting provider protocol versions for multiple major releases
- Providers built for Terraform Plugin SDK/Framework are compatible with OpenTofu
- OpenTofu maintains compatibility with the Terraform registry protocol

## What May Change

### Experimental Features

Features marked `experimental` may change or be removed:

```hcl
# Experimental features must be opted into explicitly
terraform {
  experiments = [module_variable_optional_attrs]  # now stable in 1.3+
}
```

### Deprecated Features

Deprecated features give advance notice before removal:
- Deprecation warning in current version
- Feature removed in next major version

```bash
# Check for deprecation warnings in your plan output
tofu plan 2>&1 | grep -i "deprecated\|will be removed"
```

## Compatibility with Terraform

OpenTofu maintains compatibility with Terraform 1.x configurations:

```hcl
# Configurations using `terraform {}` block work with OpenTofu
terraform {
  required_version = ">= 1.5.0"  # works with both Terraform 1.x and OpenTofu 1.x
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## The `.terraform.lock.hcl` Commitment

The dependency lock file format is stable:
- Lock files created by older versions work with newer versions
- Lock files can be committed to version control safely
- Hash algorithms in lock files are guaranteed to remain valid

```bash
# The lock file format is stable across versions
git add .terraform.lock.hcl
git commit -m "chore: update provider lock file"
```

## Testing Against Compatibility Promises

```bash
# Pin your required_version to enforce the compatibility boundary
terraform {
  required_version = ">= 1.8.0, < 2.0.0"
}

# Test with both minimum and latest version in CI
strategy:
  matrix:
    tofu-version: ["1.8.0", "1.9.0", "latest"]
```

## Summary

OpenTofu's compatibility promises cover HCL configuration syntax, state format, provider protocols, and the lock file format within major version boundaries. Experimental features and deprecated items are the exceptions. Pinning `required_version` to a major version range and testing with multiple versions protects against unexpected breakage.
