# How to Use the setintersection Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the setintersection function in OpenTofu to find common elements between multiple sets for permission and resource overlap analysis.

## Introduction

The `setintersection` function in OpenTofu returns a set containing only the elements that appear in ALL of the provided sets. It is useful for finding common permissions, shared resources, and overlapping configurations between different groups.

## Syntax

```hcl
setintersection(sets...)
```

- Accepts two or more sets
- Returns a set with elements present in all inputs

## Basic Examples

```hcl
output "intersection" {
  value = setintersection(["a", "b", "c"], ["b", "c", "d"])
  # Returns toset(["b", "c"])
}

output "no_overlap" {
  value = setintersection(["a", "b"], ["c", "d"])
  # Returns toset([])
}
```

## Practical Use Cases

### Finding Shared Permissions

```hcl
variable "team_a_perms" {
  type    = list(string)
  default = ["s3:GetObject", "s3:PutObject", "ec2:DescribeInstances"]
}

variable "team_b_perms" {
  type    = list(string)
  default = ["s3:GetObject", "ec2:DescribeInstances", "rds:DescribeDBInstances"]
}

locals {
  shared_perms = setintersection(toset(var.team_a_perms), toset(var.team_b_perms))
}

output "shared_permissions" {
  value = local.shared_perms
  # ["ec2:DescribeInstances", "s3:GetObject"]
}
```

### Validating Required Tags

```hcl
variable "resource_tags" {
  type = map(string)
}

locals {
  required_keys = toset(["environment", "team", "project"])
  provided_keys = toset(keys(var.resource_tags))
  present_required = setintersection(local.required_keys, local.provided_keys)
  missing_keys     = setsubtract(local.required_keys, local.present_required)
}
```

### Finding Common AZs

```hcl
variable "service_a_azs" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "service_b_azs" {
  type    = list(string)
  default = ["us-east-1b", "us-east-1c"]
}

locals {
  shared_azs = setintersection(toset(var.service_a_azs), toset(var.service_b_azs))
}

output "deployable_azs" {
  value = local.shared_azs  # ["us-east-1b", "us-east-1c"]
}
```

## Step-by-Step Usage

```bash
tofu console

> setintersection(["a", "b", "c"], ["b", "c", "d"], ["c", "d", "e"])
toset(["c"])
```

## Conclusion

The `setintersection` function finds common elements across multiple sets. Use it to identify shared permissions, common AZs, overlapping resources, and any other scenario requiring set intersection logic in OpenTofu.
