# How to Use try and can for Graceful Error Handling in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, HCL, Error Handling, try, can, Infrastructure as Code

Description: Learn how to use the `try` and `can` functions in OpenTofu to handle errors gracefully in expressions, providing fallback values and conditional logic without failing the entire plan.

## Introduction

OpenTofu's `try` and `can` functions allow you to handle potential errors in expressions without causing the entire configuration to fail. They are essential for dealing with optional attributes, nullable values, and dynamic configurations.

## The `can` Function

`can` evaluates an expression and returns `true` if it succeeds without errors, or `false` if it would fail:

```hcl
# Check if a CIDR block is valid
variable "cidr_block" {
  type = string
}

locals {
  is_valid_cidr = can(cidrhost(var.cidr_block, 0))
}

output "cidr_valid" {
  value = local.is_valid_cidr
}
```

## Using `can` in Variable Validation

```hcl
variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid IPv4 CIDR block."
  }
}
```

## The `try` Function

`try` evaluates a series of expressions and returns the first one that doesn't produce an error:

```hcl
# Use try to access a potentially null attribute
locals {
  instance_name = try(aws_instance.web[0].tags["Name"], "unnamed-instance")
}
```

## Handling Optional Map Keys

```hcl
variable "config" {
  type = map(string)
  default = {}
}

locals {
  # Return the value or a default if key doesn't exist
  timeout = try(var.config["timeout"], "30s")
  region  = try(var.config["region"], "us-east-1")
}
```

## Handling Conditional Resources

When a resource may or may not exist (e.g., count = 0):

```hcl
resource "aws_instance" "optional" {
  count = var.create_instance ? 1 : 0
  ...
}

output "instance_ip" {
  value = try(aws_instance.optional[0].public_ip, "not-created")
}
```

## Parsing Dynamic Data with try

```hcl
# Safely parse JSON that might be malformed
locals {
  tags = try(jsondecode(var.tags_json), {})
}
```

## Differences Between try and can

| Function | Returns | Use Case |
|----------|---------|----------|
| `can` | `true`/`false` | Checking if expression is valid |
| `try` | First successful value | Providing fallbacks |

## Combining try and can

```hcl
locals {
  # Check validity then use safely
  raw_ip = var.maybe_ip
  has_valid_ip = can(cidrhost("${local.raw_ip}/32", 0))
  safe_ip = try(cidrhost("${local.raw_ip}/32", 0), null)
}
```

## Common Pitfalls

Avoid using `try` to hide genuine errors — it should handle expected optional scenarios, not mask bugs:

```hcl
# Bad: masking a real configuration error
value = try(aws_resource.required.id, "")

# Good: handling a genuinely optional resource
value = try(aws_resource.optional[0].id, "")
```

## Conclusion

`try` and `can` are powerful tools for building resilient OpenTofu configurations that handle optional attributes and variable inputs gracefully. Use `can` for validation checks and `try` for providing meaningful fallback values.
