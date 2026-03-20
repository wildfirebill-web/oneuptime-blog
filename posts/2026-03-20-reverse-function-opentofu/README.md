# How to Use the reverse Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Reverse, List Functions, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the reverse function in OpenTofu to reverse the order of elements in a list.

---

`reverse()` takes a list and returns a new list with the same elements in reversed order. Note: this is for reversing lists - for reversing strings, use `strrev()`.

---

## Syntax

```hcl
reverse(list)
```

---

## Basic Examples

```hcl
locals {
  forward  = ["a", "b", "c", "d"]
  backward = reverse(local.forward)
  # ["d", "c", "b", "a"]

  numbers  = [1, 2, 3, 4, 5]
  reversed = reverse(local.numbers)
  # [5, 4, 3, 2, 1]
}
```

---

## Reversing Priority Order

```hcl
variable "priority_regions" {
  type    = list(string)
  description = "Regions in order of priority (most preferred first)"
  default = ["us-east-1", "us-west-2", "eu-west-1"]
}

locals {
  # Reverse for fallback order (least preferred first)
  fallback_regions = reverse(var.priority_regions)
  # ["eu-west-1", "us-west-2", "us-east-1"]
}
```

---

## Last Element Access Pattern

A common pattern for getting the last element of a list:

```hcl
variable "sorted_versions" {
  type    = list(string)
  default = ["v1.0", "v1.1", "v1.2", "v2.0"]
}

locals {
  # Get the latest version (last element)
  latest_version = reverse(var.sorted_versions)[0]
  # "v2.0"

  # Equivalent to:
  # element(var.sorted_versions, length(var.sorted_versions) - 1)
}
```

---

## Reversing a Stack (LIFO Order)

```hcl
variable "deployment_steps" {
  type    = list(string)
  default = ["provision", "configure", "deploy", "verify"]
}

locals {
  # Rollback steps in reverse order
  rollback_steps = reverse(var.deployment_steps)
  # ["verify", "deploy", "configure", "provision"]
}

output "rollback_procedure" {
  value = join(" → ", local.rollback_steps)
  # "verify → deploy → configure → provision"
}
```

---

## Reversing for Cleanup Order

```hcl
variable "resource_creation_order" {
  type    = list(string)
  default = ["vpc", "subnets", "security_groups", "instances", "load_balancer"]
}

locals {
  # Cleanup order is the reverse of creation order
  cleanup_order = reverse(var.resource_creation_order)
  # ["load_balancer", "instances", "security_groups", "subnets", "vpc"]
}
```

---

## reverse vs sort

```hcl
locals {
  items = ["banana", "apple", "cherry"]

  reversed = reverse(local.items)   # ["cherry", "apple", "banana"] - original order reversed
  sorted   = sort(local.items)      # ["apple", "banana", "cherry"] - alphabetically sorted
  sorted_desc = reverse(sort(local.items))  # ["cherry", "banana", "apple"] - desc sort
}
```

---

## Summary

`reverse(list)` reverses the order of elements in a list. Use it to get descending-order access to ascending-sorted lists, get the last element as `reverse(list)[0]`, create rollback or cleanup sequences that mirror creation sequences, and reverse priority lists for fallback processing. For reversing a string (not a list), use `strrev()` instead.
