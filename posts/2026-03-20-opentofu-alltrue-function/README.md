# How to Use the alltrue Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the alltrue function in OpenTofu to check that all elements in a list are true for comprehensive validation logic.

## Introduction

The `alltrue` function in OpenTofu returns `true` if all elements of a boolean list are `true`. It is commonly used in validation blocks and preconditions to check multiple conditions at once.

## Syntax

```hcl
alltrue(list)
```

- Returns `true` if all elements are `true`
- Returns `false` if any element is `false`
- An empty list returns `true`

## Basic Examples

```hcl
output "all_true" {
  value = alltrue([true, true, true])   # Returns true
}

output "has_false" {
  value = alltrue([true, false, true])  # Returns false
}

output "empty" {
  value = alltrue([])                   # Returns true
}
```

## Practical Use Cases

### Multi-Condition Variable Validation

```hcl
variable "instance_type" {
  type = string

  validation {
    condition = alltrue([
      length(var.instance_type) > 0,
      strcontains(var.instance_type, "."),
      contains(["t3", "m5", "r5", "c5"], split(".", var.instance_type)[0])
    ])
    error_message = "instance_type must be a valid AWS instance type family."
  }
}
```

### Bulk Tag Validation

```hcl
variable "resource_tags" {
  type = map(string)
}

locals {
  tag_checks = [
    length(lookup(var.resource_tags, "environment", "")) > 0,
    length(lookup(var.resource_tags, "team", "")) > 0,
    length(lookup(var.resource_tags, "project", "")) > 0
  ]

  all_required_tags_present = alltrue(local.tag_checks)
}
```

### Verifying All Subnets Are in VPC

```hcl
variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24"]
}

locals {
  all_subnets_in_vpc = alltrue([
    for cidr in var.subnet_cidrs :
    cidrcontains(var.vpc_cidr, cidr)
  ])
}
```

### Checking All Services Are Healthy

```hcl
variable "service_health_checks" {
  type = map(bool)
  default = {
    api      = true
    database = true
    cache    = true
  }
}

locals {
  all_healthy = alltrue(values(var.service_health_checks))
}

output "cluster_healthy" {
  value = local.all_healthy
}
```

## Step-by-Step Usage

```bash
tofu console

> alltrue([true, true, true])
true
> alltrue([true, false])
false
> alltrue([for x in [1, 2, 3] : x > 0])
true
```

## alltrue vs anytrue

| Function | Returns true when... |
|----------|---------------------|
| `alltrue(list)` | ALL elements are true |
| `anytrue(list)` | AT LEAST ONE element is true |

## Conclusion

The `alltrue` function enables concise multi-condition validation in OpenTofu. Use it in `validation` blocks to combine multiple conditions, in `precondition` blocks to verify multiple resource attributes simultaneously, and in `locals` to compute composite health/compliance status.
