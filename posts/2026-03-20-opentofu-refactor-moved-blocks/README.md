# How to Refactor Modules with moved Blocks in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Modules

Description: Learn how to use moved blocks in OpenTofu to safely rename or reorganize resources and modules without destroying and recreating infrastructure.

## Introduction

When you refactor infrastructure - renaming a resource, moving it into a module, or restructuring module calls - OpenTofu would normally destroy and recreate it because the state address changed. The `moved` block tells OpenTofu that an existing resource has moved to a new address, preventing unnecessary recreation.

## Basic Resource Rename

Rename a resource without destroying it:

```hcl
# Before: resource "aws_instance" "web"

# After:  resource "aws_instance" "app_server"

moved {
  from = aws_instance.web
  to   = aws_instance.app_server
}
```

After applying, the state address is updated and the `moved` block can be removed (or kept for documentation).

## Moving a Resource into a Module

When you encapsulate an existing resource inside a new module:

```hcl
# Before: resource "aws_s3_bucket" "data" lived in root module
# After:  module "storage" contains aws_s3_bucket.data

moved {
  from = aws_s3_bucket.data
  to   = module.storage.aws_s3_bucket.data
}
```

## Renaming a Module

When you rename a module block:

```hcl
# Before: module "old_vpc"
# After:  module "main_vpc"

moved {
  from = module.old_vpc
  to   = module.main_vpc
}
```

This moves all resources inside the module to their new addresses.

## Moving Resources with count

When moving counted resources:

```hcl
# Before: aws_instance.server (single resource)
# After:  aws_instance.server[0] (now using count)

moved {
  from = aws_instance.server
  to   = aws_instance.server[0]
}
```

## Moving Resources with for_each

```hcl
# Before: module.vpc[0] (count-based)
# After:  module.vpc["production"] (for_each-based)

moved {
  from = module.vpc[0]
  to   = module.vpc["production"]
}
```

## Multiple moved Blocks

You can stack multiple `moved` blocks in a single file:

```hcl
# Reorganization refactor
moved {
  from = aws_security_group.app
  to   = module.compute.aws_security_group.app
}

moved {
  from = aws_launch_template.app
  to   = module.compute.aws_launch_template.app
}

moved {
  from = aws_autoscaling_group.app
  to   = module.compute.aws_autoscaling_group.app
}
```

## Verifying the Move

```bash
# Run a plan to verify the move produces no infrastructure changes
tofu plan

# Expected output:
# Plan: 0 to add, 0 to change, 0 to destroy.
```

If the plan shows no changes, the `moved` block correctly mapped the old address to the new one.

## Keeping moved Blocks in Public Modules

If you are publishing a module and a resource address changed between versions, keep the `moved` block in the module to automatically migrate existing users' state:

```hcl
# In your module - helps existing users upgrade without recreation
moved {
  from = aws_iam_role.this
  to   = aws_iam_role.cluster
}
```

## Conclusion

The `moved` block is essential for safe infrastructure refactoring. Use it whenever you rename resources, reorganize into modules, or change from `count` to `for_each`. A plan showing zero changes confirms the move succeeded. Public module authors should ship `moved` blocks to make upgrades seamless for existing consumers.
