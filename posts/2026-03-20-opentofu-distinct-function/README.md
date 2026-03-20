# How to Use the distinct Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the distinct function in OpenTofu to remove duplicate elements from a list for deduplicating security groups, tags, and resource identifiers.

## Introduction

The `distinct` function in OpenTofu returns a new list with duplicate elements removed, preserving the order of first occurrence. It is useful for deduplicating lists that may contain repeated values from multiple sources, variables, or computed results.

## Syntax

```hcl
distinct(list)
```

- **list** — a list of values (any type)
- Returns a new list with duplicates removed
- Order of first occurrence is preserved

## Basic Examples

```hcl
output "remove_duplicates" {
  value = distinct(["a", "b", "a", "c", "b"])  # Returns ["a", "b", "c"]
}

output "numbers_distinct" {
  value = distinct([1, 2, 2, 3, 1, 4])          # Returns [1, 2, 3, 4]
}

output "already_unique" {
  value = distinct(["x", "y", "z"])             # Returns ["x", "y", "z"]
}
```

## Practical Use Cases

### Deduplicating Security Group IDs

```hcl
variable "base_security_groups" {
  type    = list(string)
  default = ["sg-001", "sg-002"]
}

variable "extra_security_groups" {
  type    = list(string)
  default = ["sg-002", "sg-003"]  # sg-002 is a duplicate
}

locals {
  # Combine and deduplicate
  all_security_groups = distinct(concat(var.base_security_groups, var.extra_security_groups))
}

resource "aws_instance" "app" {
  ami             = data.aws_ami.ubuntu.id
  instance_type   = "t3.medium"
  security_groups = local.all_security_groups  # ["sg-001", "sg-002", "sg-003"]

  tags = {
    Name = "app-server"
  }
}
```

### Deduplicating Subnet IDs

```hcl
variable "primary_subnets" {
  type    = list(string)
  default = ["subnet-a", "subnet-b"]
}

variable "dr_subnets" {
  type    = list(string)
  default = ["subnet-b", "subnet-c"]  # subnet-b appears in both
}

locals {
  all_subnets = distinct(concat(var.primary_subnets, var.dr_subnets))
}

output "unique_subnets" {
  value = local.all_subnets  # ["subnet-a", "subnet-b", "subnet-c"]
}
```

### Deduplicating Tag Values

```hcl
variable "service_tags" {
  type    = list(string)
  default = ["team:platform", "env:prod", "team:platform", "tier:backend"]
}

locals {
  unique_tags = distinct(var.service_tags)
}

output "unique_service_tags" {
  value = local.unique_tags
  # ["team:platform", "env:prod", "tier:backend"]
}
```

### Collecting Unique Regions from Resources

```hcl
variable "resource_configs" {
  type = list(object({
    name   = string
    region = string
  }))
  default = [
    { name = "api", region = "us-east-1" },
    { name = "worker", region = "us-west-2" },
    { name = "db", region = "us-east-1" },
    { name = "cache", region = "us-east-1" }
  ]
}

locals {
  # Get unique deployment regions
  unique_regions = distinct([for r in var.resource_configs : r.region])
}

output "deployment_regions" {
  value = local.unique_regions  # ["us-east-1", "us-west-2"]
}
```

### Normalizing IAM Policy Actions

```hcl
variable "required_actions" {
  type    = list(string)
  default = [
    "s3:GetObject",
    "s3:PutObject",
    "s3:GetObject",  # Duplicate
    "s3:DeleteObject"
  ]
}

locals {
  unique_actions = distinct(var.required_actions)
}

resource "aws_iam_policy" "s3" {
  name = "s3-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = local.unique_actions  # No duplicates
      Resource = "*"
    }]
  })
}
```

## Step-by-Step Usage

1. Collect the list that may contain duplicates.
2. Apply `distinct(list)`.
3. Use the deduplicated list in resource arguments.
4. Test in `tofu console`:

```bash
tofu console

> distinct(["a", "b", "a", "c"])
["a", "b", "c"]
> length(distinct([1, 1, 2, 2, 3]))
3
```

## distinct vs toset

| Function | Result | Preserves Order |
|----------|--------|----------------|
| `distinct(list)` | A list with unique elements | Yes (first occurrence) |
| `toset(list)` | A set (unordered, unique) | No |

Use `distinct` when order matters; use `toset` when you need a set type for `for_each`.

## Conclusion

The `distinct` function is a clean solution to deduplication in OpenTofu. Whenever you are concatenating lists from multiple sources — security groups, subnets, IAM actions, or tags — `distinct` ensures you end up with a clean, deduplicated result. It is often paired with `concat` to merge and deduplicate in one step.
