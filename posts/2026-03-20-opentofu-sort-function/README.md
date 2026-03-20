# How to Use the sort Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the sort function in OpenTofu to sort a list of strings in lexicographical order for consistent resource ordering and version management.

## Introduction

The `sort` function in OpenTofu sorts a list of strings in ascending lexicographical order. It ensures consistent ordering, which is important for deterministic resource configurations and version comparisons.

## Syntax

```hcl
sort(list)
```

- Accepts a list of strings
- Returns a new list sorted lexicographically (A-Z, 0-9)

## Basic Examples

```hcl
output "sorted_strings" {
  value = sort(["banana", "apple", "cherry"])  # Returns ["apple", "banana", "cherry"]
}

output "sorted_versions" {
  value = sort(["1.10.0", "1.2.0", "1.9.0"])  # Returns ["1.10.0", "1.2.0", "1.9.0"] (lexicographic!)
}
```

Note: `sort` is lexicographic, so `"1.10.0"` comes before `"1.2.0"`. For version sorting, use semantic version padding or a different comparison strategy.

## Practical Use Cases

### Consistent Tag Ordering

```hcl
variable "environment_names" {
  type    = list(string)
  default = ["prod", "dev", "staging"]
}

locals {
  sorted_envs = sort(var.environment_names)
}

output "environments_sorted" {
  value = local.sorted_envs  # ["dev", "prod", "staging"]
}
```

### Consistent Security Group Rule Ordering

```hcl
variable "allowed_cidrs" {
  type    = list(string)
  default = ["192.168.0.0/16", "10.0.0.0/8", "172.16.0.0/12"]
}

locals {
  # Sort for deterministic plan output
  sorted_cidrs = sort(var.allowed_cidrs)
}
```

### Getting First/Last Alphabetically

```hcl
variable "bucket_names" {
  type    = list(string)
  default = ["myapp-logs", "myapp-data", "myapp-archive"]
}

locals {
  sorted        = sort(var.bucket_names)
  first_alpha   = local.sorted[0]                           # "myapp-archive"
  last_alpha    = reverse(local.sorted)[0]                  # "myapp-logs"
}
```

### Ensuring Consistent for_each Keys

```hcl
variable "service_names" {
  type    = list(string)
  default = ["worker", "api", "frontend"]
}

resource "aws_iam_role" "service" {
  for_each = toset(sort(var.service_names))
  name     = "role-${each.key}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## Step-by-Step Usage

```bash
tofu console

> sort(["c", "a", "b"])
["a", "b", "c"]
> sort(["z10", "z2", "z1"])
["z1", "z10", "z2"]  # Lexicographic, not numeric!
```

## Lexicographic vs Numeric Sorting

For numeric sort, convert to zero-padded strings:

```hcl
locals {
  # Pad numbers to same length for correct lexicographic sort
  padded = [for n in [10, 2, 1, 100] : format("%04d", n)]
  sorted_padded = sort(local.padded)  # ["0001", "0002", "0010", "0100"]
}
```

## Conclusion

The `sort` function provides deterministic list ordering in OpenTofu. Use it to normalize input ordering, ensure consistent `for_each` expansion, and find alphabetically first or last elements. Remember it sorts lexicographically - for numeric sorting, use zero-padded strings.
