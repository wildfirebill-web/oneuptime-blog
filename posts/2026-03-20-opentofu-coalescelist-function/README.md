# How to Use the coalescelist Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the coalescelist function in OpenTofu to return the first non-empty list from a set of candidates for flexible default list handling.

## Introduction

The `coalescelist` function in OpenTofu returns the first non-empty list from the provided arguments. It is the list equivalent of `coalesce` and is useful for providing fallback lists when a primary list may be empty.

## Syntax

```hcl
coalescelist(list1, list2, ...)
```

- Returns the first list that is not empty
- Raises an error if all lists are empty

## Basic Examples

```hcl
output "first_non_empty" {
  value = coalescelist([], ["a", "b"], ["c"])  # Returns ["a", "b"]
}

output "first_list_wins" {
  value = coalescelist(["x"], ["y", "z"])      # Returns ["x"]
}
```

## Practical Use Cases

### Fallback Subnet Lists

```hcl
variable "custom_subnets" {
  type    = list(string)
  default = []
}

variable "default_subnets" {
  type    = list(string)
  default = ["subnet-default-1", "subnet-default-2"]
}

locals {
  active_subnets = coalescelist(var.custom_subnets, var.default_subnets)
}

resource "aws_autoscaling_group" "app" {
  vpc_zone_identifier = local.active_subnets
  min_size            = 1
  max_size            = 5
  desired_capacity    = 2

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
}
```

### Fallback Security Groups

```hcl
variable "custom_security_groups" {
  type    = list(string)
  default = []
}

locals {
  default_sgs    = [aws_security_group.default.id]
  active_sgs     = coalescelist(var.custom_security_groups, local.default_sgs)
}
```

### Optional Override Lists

```hcl
variable "override_dns_servers" {
  type    = list(string)
  default = []
}

locals {
  corporate_dns = ["8.8.8.8", "8.8.4.4"]
  dns_servers   = coalescelist(var.override_dns_servers, local.corporate_dns)
}
```

## Step-by-Step Usage

```bash
tofu console

> coalescelist([], [1, 2, 3])
[1, 2, 3]
> coalescelist([1], [2, 3])
[1]
```

## Conclusion

The `coalescelist` function enables clean fallback list logic in OpenTofu. Use it when a list input is optional and you want to automatically fall back to a default list when the primary is empty.
