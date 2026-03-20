# How to Use the length Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the length function in OpenTofu to count elements in lists, sets, maps, and characters in strings for conditional and dynamic configuration.

## Introduction

The `length` function in OpenTofu returns the number of elements in a collection (list, set, or map) or the number of characters in a string. It is one of the most commonly used functions for conditional resource creation, validation, and dynamic sizing.

## Syntax

```hcl
length(value)
```

- For strings: returns the number of Unicode characters
- For lists/sets: returns the number of elements
- For maps: returns the number of key-value pairs

## Basic Examples

```hcl
output "string_length" {
  value = length("hello")           # Returns 5
}

output "list_length" {
  value = length(["a", "b", "c"])   # Returns 3
}

output "map_length" {
  value = length({a = 1, b = 2})   # Returns 2
}

output "empty_list" {
  value = length([])               # Returns 0
}
```

## Practical Use Cases

### Conditional Resource Creation

```hcl
variable "extra_disk_sizes" {
  type    = list(number)
  default = [100, 200]
}

resource "aws_ebs_volume" "extra" {
  # Only create if extra disks are specified
  count             = length(var.extra_disk_sizes) > 0 ? length(var.extra_disk_sizes) : 0
  availability_zone = "us-east-1a"
  size              = var.extra_disk_sizes[count.index]

  tags = {
    Name = "extra-disk-${count.index + 1}"
  }
}
```

### Validating Non-Empty Lists

```hcl
variable "subnet_ids" {
  type = list(string)

  validation {
    condition     = length(var.subnet_ids) >= 2
    error_message = "At least 2 subnet IDs are required for high availability."
  }
}
```

### Dynamic Count Based on List Size

```hcl
variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8", "172.16.0.0/12"]
}

resource "aws_security_group_rule" "allow_cidrs" {
  count       = length(var.allowed_cidrs)
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = [var.allowed_cidrs[count.index]]

  security_group_id = aws_security_group.app.id
}
```

### Checking String Length

```hcl
variable "resource_prefix" {
  type = string

  validation {
    condition     = length(var.resource_prefix) >= 3 && length(var.resource_prefix) <= 20
    error_message = "resource_prefix must be between 3 and 20 characters."
  }
}
```

### Asserting Non-Empty Maps

```hcl
variable "tags" {
  type = map(string)

  validation {
    condition     = length(var.tags) > 0
    error_message = "At least one tag must be provided."
  }
}
```

## Step-by-Step Usage

1. Call `length(collection_or_string)`.
2. Use the result in `count`, `validation`, or arithmetic expressions.

```bash
tofu console

> length("hello")
5
> length([1, 2, 3])
3
> length({})
0
```

## Combining length with Other Functions

```hcl
locals {
  has_items = length(var.my_list) > 0
  last_idx  = length(var.my_list) - 1
  mid_item  = element(var.my_list, floor(length(var.my_list) / 2))
}
```

## Conclusion

The `length` function is fundamental to dynamic OpenTofu configurations. Use it to guard against empty inputs, drive `count` values, validate minimum and maximum sizes, and make conditional decisions based on collection sizes. It works uniformly across strings, lists, sets, and maps.
