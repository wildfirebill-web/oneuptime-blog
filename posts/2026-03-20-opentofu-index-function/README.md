# How to Use the index Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the index function in OpenTofu to find the position of a specific value within a list for position-based resource configuration.

## Introduction

The `index` function in OpenTofu returns the position (0-based) of the first occurrence of a given value in a list. It is the inverse of `element` and is useful when you know the value but need its position, for example to select matching elements from parallel lists.

## Syntax

```hcl
index(list, value)
```

- **list** — the list to search
- **value** — the value to find
- Returns the zero-based index of the first match
- Raises an error if the value is not found

## Basic Examples

```hcl
output "find_b" {
  value = index(["a", "b", "c"], "b")  # Returns 1
}

output "find_first" {
  value = index(["x", "y", "x"], "x")  # Returns 0 (first occurrence)
}
```

## Practical Use Cases

### Parallel List Lookup

```hcl
variable "instance_types" {
  type    = list(string)
  default = ["t3.micro", "t3.small", "t3.medium", "t3.large"]
}

variable "monthly_costs" {
  type    = list(number)
  default = [8.47, 16.94, 33.89, 67.77]
}

variable "selected_instance_type" {
  type    = string
  default = "t3.medium"
}

locals {
  idx           = index(var.instance_types, var.selected_instance_type)
  selected_cost = var.monthly_costs[local.idx]
}

output "monthly_cost" {
  value = local.selected_cost  # Returns 33.89
}
```

### Getting AZ Index

```hcl
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "target_az" {
  type    = string
  default = "us-east-1b"
}

locals {
  az_index = index(var.availability_zones, var.target_az)
}

output "az_index" {
  value = local.az_index  # Returns 1
}
```

### Subnet CIDR from AZ Position

```hcl
variable "az_list" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  for_each = toset(var.az_list)

  vpc_id            = aws_vpc.main.id
  availability_zone = each.key
  # Use AZ index to determine CIDR
  cidr_block        = "10.0.${index(var.az_list, each.key)}.0/24"

  tags = {
    Name = "public-${each.key}"
  }
}
```

## Step-by-Step Usage

1. Know what value you are looking for and in which list.
2. Call `index(list, value)`.
3. Handle potential errors with `try` if the value may not exist.

```bash
tofu console

> index(["a", "b", "c"], "b")
1
> index(["x", "y"], "z")  # Error: value not found
```

## Safe Lookup with try

```hcl
locals {
  position = try(index(var.options, var.selected), -1)
  is_valid  = local.position >= 0
}
```

## Conclusion

The `index` function is the inverse of `element` — use it when you know the value and need its position. It is particularly useful for parallel list lookups and position-based CIDR or configuration generation in OpenTofu.
