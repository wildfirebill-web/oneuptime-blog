# How to Use the slice Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the slice function in OpenTofu to extract a sub-list by start and end index for pagination and list partitioning.

## Introduction

The `slice` function in OpenTofu extracts a contiguous portion of a list by specifying start and end indices. It is useful for list partitioning, pagination, and extracting specific segments of resource collections.

## Syntax

```hcl
slice(list, startindex, endindex)
```

- **startindex** - inclusive starting index (0-based)
- **endindex** - exclusive ending index
- Returns `list[startindex..endindex-1]`

## Basic Examples

```hcl
output "middle_slice" {
  value = slice(["a", "b", "c", "d", "e"], 1, 4)  # Returns ["b", "c", "d"]
}

output "first_two" {
  value = slice([10, 20, 30, 40], 0, 2)            # Returns [10, 20]
}
```

## Practical Use Cases

### Pagination

```hcl
variable "all_items" {
  type    = list(string)
  default = ["item1", "item2", "item3", "item4", "item5", "item6"]
}

variable "page_size" {
  type    = number
  default = 2
}

variable "page_number" {
  type    = number
  default = 1  # 0-indexed
}

locals {
  start = var.page_number * var.page_size
  end   = min(local.start + var.page_size, length(var.all_items))
  page  = slice(var.all_items, local.start, local.end)
}

output "current_page" {
  value = local.page  # ["item3", "item4"] for page 1
}
```

### Limiting Resource Count

```hcl
variable "all_subnet_ids" {
  type    = list(string)
  default = ["s1", "s2", "s3", "s4", "s5"]
}

locals {
  # Only use first 3 subnets
  primary_subnets = slice(var.all_subnet_ids, 0, min(3, length(var.all_subnet_ids)))
}
```

### Getting All But First/Last

```hcl
variable "versions" {
  type    = list(string)
  default = ["v1", "v2", "v3", "v4", "v5"]
}

locals {
  # All except first (skip oldest)
  recent_versions = slice(var.versions, 1, length(var.versions))
  # All except last (skip newest/unstable)
  stable_versions = slice(var.versions, 0, length(var.versions) - 1)
}
```

## Step-by-Step Usage

```bash
tofu console

> slice([1, 2, 3, 4, 5], 1, 4)
[2, 3, 4]
> slice(["a", "b", "c"], 0, 2)
["a", "b"]
```

## Conclusion

The `slice` function provides precise list range extraction in OpenTofu. Use it for pagination, limiting resource counts, and extracting meaningful sub-lists from larger collections.
