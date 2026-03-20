# How to Use the compact Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the compact function in OpenTofu to remove null and empty string values from a list for cleaner resource configurations.

## Introduction

The `compact` function in OpenTofu removes all empty string (`""`) and `null` values from a list of strings, returning a new list with only the non-empty elements. It is useful for cleaning up optionally populated lists.

## Syntax

```hcl
compact(list)
```

- Accepts a `list(string)`
- Returns the same list with empty strings and nulls removed

## Basic Examples

```hcl
output "compact_list" {
  value = compact(["a", "", "b", null, "c"])  # Returns ["a", "b", "c"]
}

output "all_empty" {
  value = compact(["", null, ""])  # Returns []
}
```

## Practical Use Cases

### Optional Security Group Rules

```hcl
variable "vpn_cidr" {
  type    = string
  default = ""  # Optional
}

variable "office_cidr" {
  type    = string
  default = "203.0.113.0/24"
}

locals {
  raw_cidrs    = [var.vpn_cidr, var.office_cidr, "10.0.0.0/8"]
  active_cidrs = compact(local.raw_cidrs)  # Empty vpn_cidr is removed
}

resource "aws_security_group_rule" "ingress" {
  type        = "ingress"
  from_port   = 443
  to_port     = 443
  protocol    = "tcp"
  cidr_blocks = local.active_cidrs

  security_group_id = aws_security_group.app.id
}
```

### Optional Feature Flags

```hcl
variable "enable_monitoring" {
  type    = bool
  default = true
}

variable "enable_backups" {
  type    = bool
  default = false
}

locals {
  features = compact([
    var.enable_monitoring ? "monitoring" : "",
    var.enable_backups ? "backups" : "",
    "logging"  # Always enabled
  ])
}

output "active_features" {
  value = local.features  # ["monitoring", "logging"]
}
```

### Cleaning Up Optional Tag Values

```hcl
variable "optional_description" {
  type    = string
  default = ""
}

locals {
  tag_values = compact([var.optional_description, "managed-by-opentofu"])
}
```

## Step-by-Step Usage

```bash
tofu console

> compact(["a", "", "b", ""])
["a", "b"]
> compact(["", null])
[]
```

## Conclusion

The `compact` function is a quick way to remove empty strings and nulls from lists in OpenTofu. It is particularly useful when list elements are conditionally set using ternary expressions that fall back to `""`.
