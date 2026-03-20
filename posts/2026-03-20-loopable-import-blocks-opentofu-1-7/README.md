# How to Use Loopable Import Blocks Introduced in OpenTofu 1.7

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Import Blocks, OpenTofu 1.7, State Management, Infrastructure as Code

Description: Learn how to use loopable import blocks introduced in OpenTofu 1.7 to import multiple existing resources into state using for_each expressions.

## Introduction

OpenTofu 1.7 extended import blocks (introduced in 1.5) to support `for_each`, enabling bulk imports of multiple existing resources without writing one import block per resource. This dramatically simplifies importing large numbers of existing cloud resources.

## Basic Import Block (OpenTofu 1.5)

```hcl
# Single import – one block per resource

import {
  id = "my-existing-bucket"
  to = aws_s3_bucket.existing
}
```

## Loopable Import Blocks (OpenTofu 1.7)

```hcl
# Import multiple S3 buckets with a single block
locals {
  existing_buckets = {
    logs     = "company-logs-prod"
    backups  = "company-backups-prod"
    assets   = "company-assets-prod"
    archives = "company-archives-prod"
  }
}

import {
  for_each = local.existing_buckets
  id       = each.value
  to       = aws_s3_bucket.existing[each.key]
}

resource "aws_s3_bucket" "existing" {
  for_each = local.existing_buckets
  bucket   = each.value
}
```

## Importing IAM Roles

```hcl
locals {
  existing_roles = {
    developer = "developer-role"
    readonly  = "readonly-role"
    devops    = "devops-role"
    data_eng  = "data-engineering-role"
  }
}

import {
  for_each = local.existing_roles
  id       = each.value  # IAM role import ID is just the role name
  to       = aws_iam_role.existing[each.key]
}

resource "aws_iam_role" "existing" {
  for_each = local.existing_roles
  name     = each.value

  # This will be populated from state after import
  # You can also run: tofu plan -generate-config-out=generated.tf
  assume_role_policy = "{}"  # placeholder – update after import
}
```

## Generating Configuration Automatically

OpenTofu can generate the resource configuration for you.

```bash
# First, define only the import blocks
# Then run plan with -generate-config-out to create resource blocks

tofu plan -generate-config-out=imported_resources.tf

# Review the generated file
cat imported_resources.tf

# Then apply to import the resources
tofu apply
```

## Mixed Manual and Loopable Imports

```hcl
# You can mix single and loopable imports in the same configuration

# Single resource import
import {
  id = "special-bucket-name"
  to = aws_s3_bucket.special
}

# Bulk import with for_each
import {
  for_each = var.existing_security_groups
  id       = each.value.id
  to       = aws_security_group.existing[each.key]
}

resource "aws_security_group" "existing" {
  for_each    = var.existing_security_groups
  name        = each.key
  description = each.value.description
  vpc_id      = each.value.vpc_id
}
```

## Importing Nested Module Resources

```hcl
module "networking" {
  source = "./modules/networking"
  # ...
}

import {
  for_each = var.existing_subnets
  id       = each.value
  to       = module.networking.aws_subnet.existing[each.key]
}
```

## Removing Import Blocks After Import

```bash
# After successfully importing, remove the import blocks
# They are only needed for the initial import operation

# Verify the import succeeded
tofu plan  # should show no changes after removing import blocks
```

## Complete Workflow

```bash
# 1. Write import blocks with for_each
# 2. Generate resource config (optional)
tofu plan -generate-config-out=generated.tf

# 3. Review and customize generated config
# 4. Apply to import
tofu apply

# 5. Remove import blocks
# 6. Run plan to verify no changes
tofu plan
```

## Summary

Loopable import blocks in OpenTofu 1.7 make bulk resource imports practical for large-scale brownfield migrations. Combined with `tofu plan -generate-config-out`, you can import hundreds of existing resources into state and generate their HCL configuration in a single workflow, dramatically reducing the manual work of adopting OpenTofu for existing infrastructure.
