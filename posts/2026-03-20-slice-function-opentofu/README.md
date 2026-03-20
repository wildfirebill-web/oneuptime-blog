# How to Use the slice Function in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Slices, List Functions, HCL, Infrastructure as Code, DevOps

Description: Learn how to use the slice function in OpenTofu to extract a contiguous subset of elements from a list.

---

`slice()` extracts a sub-list from a list, returning elements from a starting index up to (but not including) an ending index.

---

## Syntax

```hcl
slice(list, start_index, end_index)
```

Returns elements from index `start_index` to `end_index - 1`. The end index is exclusive.

---

## Basic Examples

```hcl
locals {
  all_items = ["a", "b", "c", "d", "e", "f"]

  first_three = slice(local.all_items, 0, 3)   # ["a", "b", "c"]
  middle      = slice(local.all_items, 2, 4)   # ["c", "d"]
  last_two    = slice(local.all_items, 4, 6)   # ["e", "f"]
}
```

---

## Selecting a Subset of Availability Zones

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c", "us-east-1d"]
}

variable "az_count" {
  type    = number
  default = 2
}

locals {
  # Use only the first az_count AZs
  selected_azs = slice(var.availability_zones, 0, var.az_count)
  # ["us-east-1a", "us-east-1b"]
}

resource "aws_subnet" "app" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  availability_zone = local.selected_azs[count.index]
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
}
```

---

## Paginating Large Lists

```hcl
variable "all_resources" {
  type    = list(string)
  default = ["r1", "r2", "r3", "r4", "r5", "r6", "r7", "r8", "r9", "r10"]
}

variable "page_size" {
  type    = number
  default = 3
}

variable "page_number" {
  type    = number
  default = 0
}

locals {
  page_start = var.page_number * var.page_size
  page_end   = min(local.page_start + var.page_size, length(var.all_resources))
  page_items = slice(var.all_resources, local.page_start, local.page_end)
  # Page 0: ["r1", "r2", "r3"]
  # Page 1: ["r4", "r5", "r6"]
}
```

---

## Splitting a List in Two

```hcl
variable "instances" {
  type    = list(string)
  default = ["i-1", "i-2", "i-3", "i-4"]
}

locals {
  half = length(var.instances) / 2

  # First half for primary target group
  primary_instances  = slice(var.instances, 0, floor(local.half))

  # Second half for secondary target group
  secondary_instances = slice(var.instances, floor(local.half), length(var.instances))
}
```

---

## Taking First N Elements

```hcl
variable "regions" {
  type    = list(string)
  default = ["us-east-1", "us-west-2", "eu-west-1", "ap-southeast-1"]
}

variable "max_regions" {
  type    = number
  default = 2
}

locals {
  active_regions = slice(
    var.regions,
    0,
    min(var.max_regions, length(var.regions))
  )
  # ["us-east-1", "us-west-2"]
}
```

---

## Removing First N Elements (Rest of List)

```hcl
variable "versions" {
  type    = list(string)
  default = ["v1.0", "v1.1", "v2.0", "v2.1", "v3.0"]
}

locals {
  # Skip the first 2 (old versions), keep the rest
  recent_versions = slice(var.versions, 2, length(var.versions))
  # ["v2.0", "v2.1", "v3.0"]
}
```

---

## Summary

`slice(list, start, end)` extracts elements from `start` (inclusive) to `end` (exclusive). Use it to select the first N elements from a list, skip the first N elements, split a list in half, implement pagination, or restrict a user-provided list to a maximum size. The end index is exclusive, so use `length(list)` as the end to include the last element.
