# How to Use the coalesce Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the coalesce function in OpenTofu to return the first non-null, non-empty value from a list of arguments for flexible default handling.

## Introduction

The `coalesce` function in OpenTofu returns the first argument that is not null and not an empty string. It is similar to SQL's `COALESCE` and is used for providing fallback values in priority order.

## Syntax

```hcl
coalesce(value1, value2, ...)
```

- Returns the first non-null, non-empty-string value
- Raises an error if all values are null or empty

## Basic Examples

```hcl
output "first_non_null" {
  value = coalesce(null, "", "hello")  # Returns "hello"
}

output "first_wins" {
  value = coalesce("first", "second")  # Returns "first"
}
```

## Practical Use Cases

### Variable with Module Default

```hcl
variable "custom_ami" {
  type    = string
  default = null  # Optional
}

locals {
  ami_id = coalesce(var.custom_ami, data.aws_ami.ubuntu.id)
}
```

### Environment Override Chain

```hcl
variable "override_instance_type" {
  type    = string
  default = ""
}

variable "environment" {
  type    = string
  default = "dev"
}

locals {
  env_defaults = {
    prod = "m5.large"
    dev  = "t3.micro"
  }

  # Try override first, then env default, then hardcoded fallback
  instance_type = coalesce(
    var.override_instance_type,
    lookup(local.env_defaults, var.environment, ""),
    "t3.small"
  )
}
```

### DNS Name Fallback

```hcl
variable "custom_endpoint" {
  type    = string
  default = ""
}

locals {
  api_endpoint = coalesce(var.custom_endpoint, "api.default.example.com")
}
```

### Multi-Source Configuration

```hcl
variable "db_host_override" {
  type    = string
  default = null
}

data "aws_ssm_parameter" "db_host" {
  name = "/app/db/host"
}

locals {
  db_host = coalesce(var.db_host_override, data.aws_ssm_parameter.db_host.value)
}
```

## Step-by-Step Usage

```bash
tofu console

> coalesce(null, "", "value")
"value"
> coalesce("first", "second")
"first"
```

## coalesce vs try vs lookup

| Function | Use Case |
|----------|----------|
| `coalesce(a, b, c)` | First non-null, non-empty value |
| `try(expr, default)` | First expression that doesn't error |
| `lookup(map, key, default)` | Map key with fallback |

## Conclusion

The `coalesce` function is OpenTofu's null-coalescing operator, providing clean fallback chains for optional configuration values. Use it when you have multiple possible sources for a value and want to pick the first available one.
