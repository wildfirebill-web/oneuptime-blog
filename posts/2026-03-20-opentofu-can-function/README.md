# How to Use the can Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the can function in OpenTofu to test whether an expression evaluates without errors for safe optional value checking.

## Introduction

The `can` function in OpenTofu evaluates an expression and returns `true` if it succeeds without errors, or `false` if it would produce an error. It is useful for testing whether an optional value exists, a type conversion succeeds, or a complex expression is valid.

## Syntax

```hcl
can(expression)
```

- Returns `true` if the expression evaluates without error
- Returns `false` if the expression would raise an error
- Does NOT return the value - just a boolean

## Basic Examples

```hcl
output "valid_index" {
  value = can(["a", "b"][2])   # Returns false (out of bounds)
}

output "valid_key" {
  value = can({a = 1}["b"])    # Returns false (key not found)
}

output "valid_parse" {
  value = can(tonumber("42"))  # Returns true
}

output "invalid_parse" {
  value = can(tonumber("abc")) # Returns false
}
```

## Practical Use Cases

### Checking for Optional Nested Values

```hcl
variable "config" {
  type = any
  default = {
    database = {
      host = "localhost"
    }
    # cache section is optional
  }
}

locals {
  has_cache_config = can(var.config.cache.host)
  cache_host       = local.has_cache_config ? var.config.cache.host : "127.0.0.1"
}
```

### Validating Parseable Inputs

```hcl
variable "json_config" {
  type        = string
  description = "JSON configuration string"

  validation {
    condition     = can(jsondecode(var.json_config))
    error_message = "json_config must be a valid JSON string."
  }
}
```

### Safe Map Key Access

```hcl
variable "optional_tags" {
  type    = map(string)
  default = {}
}

locals {
  has_cost_center = can(var.optional_tags["cost-center"])
  cost_center     = local.has_cost_center ? var.optional_tags["cost-center"] : "unassigned"
}
```

### Checking if a File is Valid YAML

```hcl
variable "config_file_path" {
  type    = string
  default = "${path.module}/config.yaml"
}

locals {
  is_valid_yaml = fileexists(var.config_file_path) && can(yamldecode(file(var.config_file_path)))
}
```

## Step-by-Step Usage

```bash
tofu console

> can(tonumber("42"))
true
> can(tonumber("abc"))
false
> can(["a", "b"][5])
false
```

## can vs try

| Function | Returns | Use Case |
|----------|---------|----------|
| `can(expr)` | Boolean | Just check if expr would error |
| `try(expr, fallback)` | Value or fallback | Get value with fallback |

Prefer `try` when you need the value; use `can` when you only need to test.

## Conclusion

The `can` function is a boolean probe in OpenTofu - it tests whether an expression succeeds without actually using its value. Use it in validation blocks to verify inputs are parseable, in conditionals to check for optional nested values, and in guards before accessing potentially missing keys.
