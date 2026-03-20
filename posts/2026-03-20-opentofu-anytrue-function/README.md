# How to Use the anytrue Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the anytrue function in OpenTofu to check if at least one element in a boolean list is true for OR-based conditional logic.

## Introduction

The `anytrue` function in OpenTofu returns `true` if at least one element in a boolean list is `true`. It implements OR logic across a list, making it useful for checking if any condition from a collection holds.

## Syntax

```hcl
anytrue(list)
```

- Returns `true` if any element is `true`
- Returns `false` if all elements are `false`
- Empty list returns `false`

## Basic Examples

```hcl
output "any_true" {
  value = anytrue([false, false, true])  # Returns true
}

output "all_false" {
  value = anytrue([false, false, false]) # Returns false
}

output "empty" {
  value = anytrue([])                    # Returns false
}
```

## Practical Use Cases

### Detecting Public Access

```hcl
variable "allowed_cidrs" {
  type    = list(string)
  default = ["10.0.0.0/8", "0.0.0.0/0"]
}

locals {
  has_public_access = anytrue([
    for cidr in var.allowed_cidrs :
    cidr == "0.0.0.0/0" || cidr == "::/0"
  ])
}

output "is_publicly_accessible" {
  value = local.has_public_access  # true
}
```

### Checking If Any Feature Is Enabled

```hcl
variable "feature_flags" {
  type = map(bool)
  default = {
    feature_a = false
    feature_b = true
    feature_c = false
  }
}

locals {
  any_feature_enabled = anytrue(values(var.feature_flags))
}

resource "aws_cloudwatch_log_group" "features" {
  count             = local.any_feature_enabled ? 1 : 0
  name              = "/app/features"
  retention_in_days = 7
}
```

### Compliance Check (Any Violation)

```hcl
variable "security_groups" {
  type = list(object({
    id   = string
    open = bool
  }))
}

locals {
  has_open_sg = anytrue([for sg in var.security_groups : sg.open])
}

resource "null_resource" "security_check" {
  lifecycle {
    precondition {
      condition     = !local.has_open_sg
      error_message = "One or more security groups have unrestricted access."
    }
  }
}
```

## Step-by-Step Usage

```bash
tofu console

> anytrue([false, true, false])
true
> anytrue([for x in [2, 4, 6] : x > 5])
true
```

## Conclusion

The `anytrue` function implements OR logic across a list of boolean values in OpenTofu. Use it to detect if any item in a collection matches a condition - for security checks, feature detection, compliance validation, and conditional resource creation.
