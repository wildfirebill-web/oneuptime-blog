# How to Use can Function for Value Validation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Can Function, Validation, HCL, Infrastructure as Code

Description: Learn how to use OpenTofu's can function to test whether expressions are valid and build conditional logic based on value presence or type.

The `can` function evaluates an expression and returns `true` if the expression succeeds without an error, or `false` if it would produce an error. Unlike `try`, it does not return the value - it returns a boolean. This makes it ideal for conditional logic and custom variable validation.

## Basic Syntax

```hcl
# Returns true if the expression is valid, false if it produces an error

can(expression)
```

## Use Case 1: Checking if a Map Key Exists

```hcl
variable "feature_flags" {
  type = map(bool)
  default = {
    enable_monitoring = true
    # "enable_tracing" is not present
  }
}

locals {
  # Check if the key exists before accessing it
  tracing_enabled = can(var.feature_flags["enable_tracing"]) ? var.feature_flags["enable_tracing"] : false
}
```

## Use Case 2: Validating Variable Format

Use `can` inside a `validation` block to check that a value matches an expected format:

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment (must be dev, staging, or production)"

  validation {
    # can() returns true only if the regex matches
    condition     = can(regex("^(dev|staging|production)$", var.environment))
    error_message = "environment must be one of: dev, staging, production"
  }
}

variable "cidr_block" {
  type        = string
  description = "VPC CIDR block in CIDR notation"

  validation {
    # Validate CIDR format using cidrhost - errors if format is invalid
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "cidr_block must be a valid CIDR notation (e.g., 10.0.0.0/16)"
  }
}
```

## Use Case 3: Checking Numeric Convertibility

```hcl
variable "port" {
  type = string
}

locals {
  # Check if the string can be converted to a number
  is_numeric_port = can(tonumber(var.port))
  port_number     = can(tonumber(var.port)) ? tonumber(var.port) : 80
}
```

## Use Case 4: Conditional Resource Behavior Based on Optional Attributes

```hcl
variable "db_config" {
  type = object({
    host            = string
    read_replica    = optional(string)
  })
}

locals {
  # Only enable read routing if a read replica is configured
  use_read_replica = can(var.db_config.read_replica) && var.db_config.read_replica != null
}

resource "aws_db_proxy_endpoint" "read" {
  count = local.use_read_replica ? 1 : 0
  # ...
}
```

## Use Case 5: Validating ARN Format

```hcl
variable "kms_key_arn" {
  type        = string
  description = "ARN of the KMS key for encryption"

  validation {
    # An ARN must match this pattern
    condition     = can(regex("^arn:aws:kms:[a-z0-9-]+:[0-9]{12}:key/[a-f0-9-]+$", var.kms_key_arn))
    error_message = "kms_key_arn must be a valid KMS key ARN."
  }
}
```

## Combining can and try

`can` and `try` are complementary:

```hcl
locals {
  # Use can to check, try to get the value safely
  has_custom_domain = can(var.config["custom_domain"])
  custom_domain     = try(var.config["custom_domain"], "")
}
```

Or more concisely with just `try`:

```hcl
locals {
  custom_domain = try(var.config["custom_domain"], "")
}
```

Use `can` when you need a boolean condition; use `try` when you need the value with a fallback.

## Conclusion

The `can` function is the right tool when you need to test whether an expression is valid before acting on it. Its primary uses are in `validation` blocks for input variable checking and in conditional logic that depends on optional attribute presence. Combine it with `try` for complete defensive value handling patterns.
