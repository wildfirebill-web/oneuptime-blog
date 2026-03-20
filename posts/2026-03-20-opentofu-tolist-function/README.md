# How to Use the tolist Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the tolist function in OpenTofu to convert sets and tuples to lists for ordered, indexable collection operations.

## Introduction

The `tolist` function in OpenTofu converts a set or tuple to a list. Lists are ordered and indexable, while sets are unordered and cannot be indexed directly. Converting to a list is necessary when you need to access elements by position or use list-specific functions.

## Syntax

```hcl
tolist(value)
```

- Converts sets and tuples to lists
- Sets are sorted alphabetically/numerically when converted
- Returns a list of the same type

## Basic Examples

```hcl
output "set_to_list" {
  value = tolist(toset(["c", "a", "b"]))
  # Returns ["a", "b", "c"] (sorted)
}

output "already_list" {
  value = tolist(["x", "y", "z"])
  # Returns ["x", "y", "z"] (unchanged)
}
```

## Practical Use Cases

### Converting fileset Output for Indexing

```hcl
locals {
  config_files = tolist(fileset("${path.module}/configs", "*.json"))
  first_config  = local.config_files[0]   # Now indexable
  last_config   = reverse(local.config_files)[0]
}
```

### Converting for_each Keys to List

```hcl
resource "aws_iam_role" "services" {
  for_each = toset(["api", "worker", "scheduler"])
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

locals {
  # Convert set to indexable list
  role_names = tolist(toset(keys(aws_iam_role.services)))
}

output "first_role" {
  value = local.role_names[0]
}
```

### Enabling slice on fileset

```hcl
locals {
  all_files    = tolist(fileset("${path.module}/data", "*.csv"))
  # Take only the first 5 files
  first_5_files = slice(local.all_files, 0, min(5, length(local.all_files)))
}
```

## Step-by-Step Usage

```bash
tofu console

> tolist(toset([3, 1, 2]))
[1, 2, 3]
> tolist(["a", "b"])[0]
"a"
```

## Conclusion

The `tolist` function enables list operations on sets in OpenTofu. Use it when you need to index, slice, or apply list-specific functions to set values. Remember that converting a set to list sorts the elements.
