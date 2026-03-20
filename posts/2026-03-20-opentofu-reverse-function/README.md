# How to Use the reverse Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the reverse function in OpenTofu to invert the order of elements in a list for priority reordering and last-element access.

## Introduction

The `reverse` function in OpenTofu returns a new list with the elements in reversed order. It is useful for priority reordering, getting the last element, and processing lists in reverse.

## Syntax

```hcl
reverse(list)
```

## Basic Examples

```hcl
output "reversed" {
  value = reverse(["a", "b", "c"])  # Returns ["c", "b", "a"]
}

output "last_element" {
  value = reverse(["a", "b", "c"])[0]  # Returns "c"
}
```

## Practical Use Cases

### Getting the Latest Version

```hcl
variable "available_versions" {
  type    = list(string)
  default = ["1.0.0", "1.1.0", "1.2.0", "2.0.0"]
}

locals {
  sorted_versions = sort(var.available_versions)
  latest_version  = reverse(local.sorted_versions)[0]
}

output "latest" {
  value = local.latest_version  # "2.0.0"
}
```

### Reversing Priority Lists

```hcl
variable "fallback_regions" {
  type    = list(string)
  default = ["eu-west-1", "us-west-2", "us-east-1"]
}

locals {
  # us-east-1 is highest priority (try it first)
  priority_order = reverse(var.fallback_regions)
}

output "try_first" {
  value = local.priority_order[0]  # "us-east-1"
}
```

### Reverse Iterating with count

```hcl
variable "items" {
  type    = list(string)
  default = ["deploy", "test", "build"]
}

locals {
  reversed_items = reverse(var.items)
  # ["build", "test", "deploy"]
}
```

## Step-by-Step Usage

```bash
tofu console

> reverse([1, 2, 3, 4])
[4, 3, 2, 1]
> reverse(["a", "b", "c"])[0]
"c"
```

## Conclusion

The `reverse` function is a simple but useful list utility in OpenTofu. Its most common use is accessing the last element of a sorted list (e.g., latest version) and reversing priority order for ordered fallback logic.
