# How to Handle Provider Breaking Changes in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Upgrades, Breaking Changes, Best Practices, Infrastructure as Code

Description: Learn how to safely upgrade providers with breaking changes in OpenTofu by reviewing changelogs, using version constraints, and performing incremental migrations.

## Introduction

Provider major version upgrades often contain breaking changes — renamed resources, changed attribute types, removed arguments, or different import formats. Without careful planning, upgrading providers can cause unexpected plan changes or apply failures. This guide covers a safe upgrade process.

## Pinning Provider Versions

Use version constraints to control when upgrades happen.

```hcl
# versions.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"   # allow patch and minor upgrades within 5.x
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "= 3.99.0"  # pin exactly to a specific version
    }
    google = {
      source  = "hashicorp/google"
      version = ">= 5.0, < 6.0"  # allow 5.x but not 6.x
    }
  }
}
```

## Reviewing the Changelog Before Upgrading

```bash
# Check what changed between versions using the GitHub releases page
# or provider changelog

# AWS provider changelog:
# https://github.com/hashicorp/terraform-provider-aws/blob/main/CHANGELOG.md

# Example: AWS provider 4.x -> 5.x breaking changes
# - aws_s3_bucket split into separate resources
# - aws_db_instance skip_final_snapshot default changed
# - aws_security_group default egress rule removed

# Search changelog for breaking changes
curl -s https://raw.githubusercontent.com/hashicorp/terraform-provider-aws/main/CHANGELOG.md \
  | grep -A 5 "BREAKING CHANGE\|v5.0"
```

## Safe Upgrade Process

```bash
# Step 1: Check your current lock file
cat .terraform.lock.hcl

# Step 2: Update version constraint to allow new version
# versions.tf: version = "~> 5.0" -> "~> 5.31.0"

# Step 3: Update providers without applying
tofu init -upgrade

# Step 4: Run plan and review carefully for unexpected changes
tofu plan -out=upgrade_plan.tfplan 2>&1 | tee upgrade_plan.txt

# Step 5: Review the plan output
grep -E "must be replaced|will be destroyed|changed" upgrade_plan.txt

# Step 6: Apply in non-prod first
ENVIRONMENT=dev tofu apply upgrade_plan.tfplan
```

## Handling AWS Provider 4.x to 5.x S3 Changes

```hcl
# Old (provider 4.x): all S3 config in one resource
resource "aws_s3_bucket" "old" {
  bucket        = "my-bucket"
  acl           = "private"              # removed in 5.x
  force_destroy = true

  versioning {                           # removed in 5.x
    enabled = true
  }

  server_side_encryption_configuration { # removed in 5.x
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

# New (provider 5.x): separate resources for each setting
resource "aws_s3_bucket" "new" {
  bucket        = "my-bucket"
  force_destroy = true
}

resource "aws_s3_bucket_versioning" "new" {
  bucket = aws_s3_bucket.new.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "new" {
  bucket = aws_s3_bucket.new.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

## Using moved Blocks for Renamed Resources

```hcl
# If a provider renames a resource type, use moved to preserve state
moved {
  from = aws_s3_bucket_acl.old_name
  to   = aws_s3_bucket_ownership_controls.new_name
}
```

## Summary

Provider breaking changes require a methodical approach: pin versions with constraints, review changelogs before upgrading, run `tofu plan` after upgrading, and apply to non-production first. Use `moved` blocks to handle resource renames without recreation, and split monolithic resources into separate resources when providers decompose them.
