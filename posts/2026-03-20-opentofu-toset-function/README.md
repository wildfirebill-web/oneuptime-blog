# How to Use the toset Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the toset function in OpenTofu to convert lists to sets for deduplication and use with for_each resource iteration.

## Introduction

The `toset` function in OpenTofu converts a list to a set, removing duplicates and eliminating ordering guarantees. Sets are the required type for `for_each` when you want to iterate over a collection of simple values, making `toset` one of the most commonly used type conversion functions.

## Syntax

```hcl
toset(list)
```

- Converts a list to a set
- Removes duplicates
- Returns an unordered collection

## Basic Examples

```hcl
output "dedup_set" {
  value = toset(["a", "b", "a", "c"])
  # Returns toset(["a", "b", "c"])
}

output "number_set" {
  value = toset([1, 2, 2, 3])
  # Returns toset([1, 2, 3])
}
```

## Practical Use Cases

### for_each with String Lists

```hcl
variable "environments" {
  type    = list(string)
  default = ["dev", "staging", "prod"]
}

resource "aws_s3_bucket" "env_buckets" {
  for_each = toset(var.environments)
  bucket   = "myapp-${each.key}-data"

  tags = {
    Environment = each.key
  }
}
```

### for_each with IAM Users

```hcl
variable "developers" {
  type    = list(string)
  default = ["alice", "bob", "carol"]
}

resource "aws_iam_user" "devs" {
  for_each = toset(var.developers)
  name     = each.key

  tags = {
    Team = "engineering"
  }
}

resource "aws_iam_group_membership" "dev_team" {
  name  = "dev-team-membership"
  group = aws_iam_group.developers.name
  users = [for u in aws_iam_user.devs : u.name]
}
```

### Deduplicating Before for_each

```hcl
variable "regions" {
  type    = list(string)
  # Might have duplicates from various sources
  default = ["us-east-1", "us-west-2", "us-east-1"]
}

resource "aws_iam_role" "regional_roles" {
  for_each = toset(var.regions)
  name     = "app-role-${each.key}"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "lambda.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

### Security Group Rules

```hcl
variable "allowed_ports" {
  type    = list(number)
  default = [80, 443, 8080, 80]  # 80 duplicated
}

resource "aws_security_group_rule" "ingress" {
  for_each = toset([for p in var.allowed_ports : tostring(p)])

  type        = "ingress"
  from_port   = tonumber(each.key)
  to_port     = tonumber(each.key)
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]

  security_group_id = aws_security_group.app.id
}
```

## Step-by-Step Usage

```bash
tofu console

> toset(["a", "b", "a"])
toset(["a", "b"])
> length(toset(["x", "x", "y"]))
2
```

## toset vs distinct

| Function | Returns | Maintains Order |
|----------|---------|----------------|
| `distinct(list)` | List (ordered) | Yes |
| `toset(list)` | Set (unordered) | No |

Use `toset` for `for_each`; use `distinct` when order matters.

## Conclusion

The `toset` function is essential in OpenTofu for two use cases: deduplicating lists and creating sets for `for_each`. Whenever you want `for_each` to iterate over a list of string values, wrapping it with `toset` is the standard pattern.
