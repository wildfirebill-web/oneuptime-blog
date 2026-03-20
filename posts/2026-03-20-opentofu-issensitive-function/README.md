# How to Use the issensitive Function in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to use the issensitive function in OpenTofu to check whether a value is marked as sensitive for conditional handling of secret data.

## Introduction

The `issensitive` function in OpenTofu returns `true` if a value is marked as sensitive, and `false` otherwise. It is useful for debugging sensitive value propagation and for conditional logic that needs to behave differently based on whether a value is a secret.

## Syntax

```hcl
issensitive(value)
```

- Returns `true` if the value is marked sensitive
- Returns `false` if the value is not sensitive

## Basic Examples

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}

variable "db_host" {
  type    = string
  default = "localhost"
}

output "password_is_sensitive" {
  value = issensitive(var.db_password)  # Returns true
}

output "host_is_sensitive" {
  value = issensitive(var.db_host)      # Returns false
}
```

## Practical Use Cases

### Validation That Secrets Are Marked Sensitive

```hcl
variable "api_key" {
  type = string
}

output "api_key_check" {
  value = issensitive(var.api_key)
  # Use this in tofu console to verify sensitivity marking
}
```

### Debugging Sensitive Value Propagation

```hcl
locals {
  derived_value = "${var.username}:${var.password}"
}

output "is_derived_sensitive" {
  # Check if sensitivity propagated from var.password
  value = issensitive(local.derived_value)  # Should be true
}
```

### Conditional Handling Based on Sensitivity

```hcl
variable "config_value" {
  type = string
}

locals {
  # Log differently based on whether value is sensitive
  log_value = issensitive(var.config_value) ? "[REDACTED]" : var.config_value
}

output "config_for_logging" {
  value = local.log_value
}
```

### Module Input Validation

```hcl
variable "credentials" {
  type = object({
    username = string
    password = string
  })
}

locals {
  # Verify the password field is protected
  password_protected = issensitive(var.credentials.password)
}

resource "null_resource" "cred_check" {
  lifecycle {
    precondition {
      condition     = local.password_protected
      error_message = "credentials.password must be a sensitive variable."
    }
  }
}
```

## Step-by-Step Usage

```bash
tofu console

> issensitive(var.my_password)
true
> issensitive("plain text")
false
> issensitive(sensitive("now sensitive"))
true
```

## Sensitive Propagation Rules

Sensitivity propagates through expressions:

```hcl
locals {
  sensitive_var = sensitive("secret")

  # These all become sensitive:
  concat_result = "${local.sensitive_var}-suffix"
  list_result   = [local.sensitive_var, "other"]
  map_result    = { key = local.sensitive_var }
}
```

## When to Use issensitive

- Debugging: verify that sensitive data is properly marked
- Testing: write checks that validate sensitivity propagation
- Documentation: confirm sensitivity expectations in outputs

## Conclusion

The `issensitive` function is primarily a debugging and testing tool in OpenTofu. Use it in `tofu console` to verify that sensitive values are properly marked, in preconditions to enforce that certain inputs must be sensitive, and in logging helpers that need to redact sensitive data conditionally.
