# How to Use the setsubtract Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the setsubtract function in OpenTofu to remove elements from a set that exist in another set for exclusion-based filtering.

## Introduction

The `setsubtract` function in OpenTofu returns a set containing elements from the first set that are NOT present in the second set. It implements set difference (A minus B) and is useful for exclusion-based filtering, computing missing elements, and removing unwanted values from collections.

## Syntax

```hcl
setsubtract(a, b)
```

- Returns elements in `a` that are not in `b`
- The order of arguments matters: `setsubtract(a, b) ≠ setsubtract(b, a)`

## Basic Examples

```hcl
output "difference" {
  value = setsubtract(["a", "b", "c", "d"], ["b", "d"])
  # Returns toset(["a", "c"])
}

output "empty_result" {
  value = setsubtract(["a", "b"], ["a", "b", "c"])
  # Returns toset([])
}
```

## Practical Use Cases

### Finding Missing Required Tags

```hcl
variable "resource_tags" {
  type    = map(string)
  default = {
    environment = "prod"
    team        = "platform"
  }
}

locals {
  required_keys = toset(["environment", "team", "project", "cost-center"])
  provided_keys = toset(keys(var.resource_tags))
  missing_keys  = setsubtract(local.required_keys, local.provided_keys)
}

resource "null_resource" "tag_check" {
  lifecycle {
    precondition {
      condition     = length(local.missing_keys) == 0
      error_message = "Missing required tags: ${join(", ", local.missing_keys)}"
    }
  }
}
```

### Excluding Reserved Regions

```hcl
variable "all_regions" {
  type    = list(string)
  default = ["us-east-1", "us-west-2", "eu-west-1", "cn-north-1", "us-gov-west-1"]
}

locals {
  restricted_regions  = toset(["cn-north-1", "us-gov-west-1"])
  deployable_regions  = setsubtract(toset(var.all_regions), local.restricted_regions)
}

output "allowed_regions" {
  value = local.deployable_regions
  # ["eu-west-1", "us-east-1", "us-west-2"]
}
```

### Permission Revocation

```hcl
variable "current_permissions" {
  type    = list(string)
  default = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"]
}

variable "revoked_permissions" {
  type    = list(string)
  default = ["s3:DeleteObject"]
}

locals {
  active_permissions = setsubtract(
    toset(var.current_permissions),
    toset(var.revoked_permissions)
  )
}

output "active_perms" {
  value = local.active_permissions
  # ["s3:GetObject", "s3:ListBucket", "s3:PutObject"]
}
```

## Step-by-Step Usage

```bash
tofu console

> setsubtract(["a", "b", "c"], ["b"])
toset(["a", "c"])
> setsubtract(["a", "b"], ["a", "b"])
toset([])
```

## Conclusion

The `setsubtract` function implements exclusion-based filtering in OpenTofu. Use it to find missing required elements, remove restricted options, revoke permissions, and compute the difference between two collections.
