# How to Report Bugs in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Bug Reporting, Open Source, Community, GitHub, Best Practice

Description: Learn how to write effective bug reports for OpenTofu that help maintainers reproduce and fix issues quickly.

## Introduction

A well-written bug report dramatically increases the chance of your issue being fixed quickly. Maintainers need to reproduce the problem, understand its impact, and identify the cause - all from your report. This guide shows how to write effective OpenTofu bug reports.

## Before Filing a Bug

```bash
# 1. Check if the bug is already reported

# Search GitHub issues: https://github.com/opentofu/opentofu/issues

# 2. Verify you're on the latest version
tofu version

# 3. Check if the bug exists on the latest release
# Download from: https://github.com/opentofu/opentofu/releases

# 4. Check the provider version – sometimes bugs are in providers
cat .terraform.lock.hcl
```

## Minimal Reproducible Example

The most important part of a bug report is a minimal config that reproduces the issue.

```hcl
# minimal_repro/main.tf – stripped down to only what's needed to show the bug

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "5.31.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  # Use mock credentials for bugs that don't require real AWS
  access_key = "mock_access_key"
  secret_key = "mock_secret_key"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    sts = "http://localhost:4566"  # LocalStack for full isolation
  }
}

# This is the minimal config that triggers the bug
variable "items" {
  default = {
    a = { name = "test", value = null }
  }
}

resource "aws_s3_bucket" "test" {
  for_each = var.items
  bucket   = each.value.name  # BUG: panics when value contains null
}
```

## Collecting Debug Information

```bash
# Enable debug logging to capture the full error
TF_LOG=DEBUG tofu plan 2>&1 | tee debug.log

# Show version information
tofu version > version_info.txt

# Show provider versions
tofu providers >> version_info.txt

# On macOS/Linux: capture OS info
uname -a >> version_info.txt
go version >> version_info.txt 2>/dev/null || true
```

## Bug Report Template

````markdown
## Bug Description
<!-- A clear, concise description of the bug -->

tofu plan panics with a nil pointer dereference when `for_each` is used with
an object that contains a null attribute value.

## Affected Versions
- OpenTofu: 1.9.0
- Provider: hashicorp/aws 5.31.0
- OS: macOS 14.3 (arm64)

## Steps to Reproduce

1. Create the following configuration:

```hcl
variable "items" {
  default = {
    "a" = { name = "test", value = null }
  }
}

resource "aws_s3_bucket" "test" {
  for_each = var.items
  bucket   = each.value.name
}
```

2. Run `tofu plan`
3. Observe the panic

## Expected Behavior
OpenTofu should produce a valid plan showing resources to create.

## Actual Behavior
```text
panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1]
goroutine 1 [running]:
github.com/opentofu/opentofu/internal/lang.evaluateForEachExpression(...)
```

## Minimal Reproducible Example
<!-- Attach or paste the minimal config above -->

## Additional Context
- This worked in OpenTofu 1.8.5
- Does not occur when null values are absent
- Debug log attached: debug.log
````

## Where to File the Bug

- **Core OpenTofu bugs**: https://github.com/opentofu/opentofu/issues/new/choose
- **Provider bugs**: the provider's own repository (e.g., `hashicorp/terraform-provider-aws`)
- **Registry bugs**: https://github.com/opentofu/registry/issues

## Summary

Effective bug reports include the exact versions involved, a minimal reproducible configuration, the expected versus actual behavior, and debug logs. The investment in creating a good bug report pays off in faster resolution - maintainers can immediately reproduce and fix the issue rather than asking for more information.
