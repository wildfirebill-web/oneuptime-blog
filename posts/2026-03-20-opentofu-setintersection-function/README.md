# How to Use the setintersection Function in OpenTofu - Function

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Functions, Collections

Description: Learn how to use the setintersection function in OpenTofu to find common elements between two or more sets for tag filtering and permission overlap detection.

## What is setintersection?

The `setintersection` function takes two or more sets and returns a new set containing only the elements that appear in **all** of the input sets. This is the mathematical intersection operation applied to OpenTofu sets.

## Syntax

```hcl
setintersection(set1, set2, ...)
```

Returns a set containing elements present in every input set.

## Basic Usage

```hcl
locals {
  set_a = toset(["a", "b", "c", "d"])
  set_b = toset(["b", "c", "e"])

  # Elements in both sets
  common = setintersection(local.set_a, local.set_b)
  # Result: toset(["b", "c"])
}
```

## Finding Common Security Groups

```hcl
variable "web_sg_ids" {
  type    = set(string)
  default = ["sg-111", "sg-222", "sg-333"]
}

variable "api_sg_ids" {
  type    = set(string)
  default = ["sg-222", "sg-333", "sg-444"]
}

locals {
  # Find security groups that both the web and API tiers share
  shared_sgs = setintersection(var.web_sg_ids, var.api_sg_ids)
  # Result: toset(["sg-222", "sg-333"])
}

output "shared_security_groups" {
  value = local.shared_sgs
}
```

## Checking Permission Overlap

```hcl
variable "required_permissions" {
  type    = set(string)
  default = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
}

variable "granted_permissions" {
  type    = set(string)
  default = ["s3:GetObject", "s3:ListBucket", "s3:PutObject"]
}

locals {
  # Find which required permissions are actually granted
  granted_required = setintersection(var.required_permissions, var.granted_permissions)
  # ["s3:GetObject", "s3:PutObject"]

  # Find required permissions that are NOT yet granted
  missing_permissions = setsubtract(var.required_permissions, local.granted_required)
  # ["s3:DeleteObject"]
}

output "permission_audit" {
  value = {
    granted  = local.granted_required
    missing  = local.missing_permissions
  }
}
```

## Multi-Set Intersection

```hcl
variable "team_a_regions" {
  type    = set(string)
  default = ["us-east-1", "us-west-2", "eu-west-1"]
}

variable "team_b_regions" {
  type    = set(string)
  default = ["us-east-1", "ap-southeast-1", "eu-west-1"]
}

variable "team_c_regions" {
  type    = set(string)
  default = ["us-east-1", "eu-west-1", "sa-east-1"]
}

locals {
  # Regions where all three teams operate
  common_regions = setintersection(
    var.team_a_regions,
    var.team_b_regions,
    var.team_c_regions
  )
  # Result: toset(["us-east-1", "eu-west-1"])
}
```

## Validating Tag Overlap

```hcl
variable "required_tags" {
  type    = set(string)
  default = ["Environment", "Team", "CostCenter", "Project"]
}

variable "applied_tags" {
  type    = map(string)
  default = {
    Environment = "production"
    Team        = "platform"
    Name        = "web-server"
  }
}

locals {
  applied_tag_keys = toset(keys(var.applied_tags))
  present_required = setintersection(var.required_tags, local.applied_tag_keys)
  missing_tags     = setsubtract(var.required_tags, local.applied_tag_keys)
}

output "tagging_compliance" {
  value = {
    present = local.present_required
    missing = local.missing_tags
  }
}
```

## Important Notes

- All arguments must be sets. Convert lists to sets with `toset()` before using `setintersection`.
- The result is always a set (unordered, with no duplicates).
- If any input set is empty, the intersection is empty.
- For list operations, convert with `toset()` first and back with `tolist()` if ordering matters.

## Conclusion

The `setintersection` function is a powerful tool for finding common elements across multiple sets of values. Whether you are auditing permissions, finding shared resources, or validating tag compliance, `setintersection` provides a clean, declarative way to compute overlaps in your infrastructure configuration.
