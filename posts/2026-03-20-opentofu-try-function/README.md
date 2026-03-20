# How to Create Optional Resource Attributes with try in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Try, Optional Attributes, Error Handling, Null Safety

Description: Learn how to use the try() function in OpenTofu to safely access potentially null or missing values and provide fallback defaults for optional configuration.

## Overview

`try(expression, fallback)` evaluates an expression and returns the fallback if the expression would produce an error. This is essential for safely accessing optional nested attributes, handling null values, and providing defaults when configuration is incomplete.

## Step 1: Safe Attribute Access

```hcl
# main.tf - try() for safe null handling

variable "config" {
  type = object({
    database = object({
      host     = string
      port     = optional(number, 5432)
      ssl_mode = optional(string)
    })
    cache = optional(object({
      host = string
      port = number
    }))
  })
}

locals {
  # Safe access to optional nested attributes
  db_port    = try(var.config.database.port, 5432)
  db_ssl     = try(var.config.database.ssl_mode, "require")

  # Cache is optional - try returns null if not configured
  cache_host = try(var.config.cache.host, null)
  cache_port = try(var.config.cache.port, 6379)

  # Only create cache resources if cache is configured
  use_cache = local.cache_host != null
}
```

## Step 2: try with Data Sources

```hcl
# Gracefully handle missing data source results
data "aws_secretsmanager_secret" "db_password" {
  name = "/app/${var.environment}/db-password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

# Fall back to variable if secret doesn't exist
locals {
  db_password = try(
    data.aws_secretsmanager_secret_version.db_password.secret_string,
    var.db_password_fallback
  )
}
```

## Step 3: try for Multiple Type Attempts

```hcl
# try() evaluates each expression until one succeeds
variable "instance_config" {
  type = any  # Could be string or object
}

locals {
  # Handle both formats: string "t3.large" or object {type = "t3.large", count = 3}
  instance_type = try(
    var.instance_config.type,      # Try object format first
    tostring(var.instance_config),  # Fall back to string format
    "t3.large"                       # Final default
  )

  instance_count = try(
    var.instance_config.count,
    1
  )
}
```

## Step 4: Practical try Patterns

```hcl
# Pattern 1: Optional map lookup with default
variable "region_configs" {
  type = map(object({
    instance_type = string
    az_count      = number
  }))

  default = {
    "us-east-1" = { instance_type = "m5.xlarge", az_count = 3 }
  }
}

locals {
  # Get region config or use defaults
  current_config = try(
    var.region_configs[var.region],
    { instance_type = "t3.large", az_count = 2 }
  )
}

# Pattern 2: Boolean from string attribute
variable "features" {
  type = map(string)
  default = {}
}

locals {
  # Safely convert string "true"/"false" to boolean
  feature_a_enabled = try(tobool(var.features["feature_a"]), false)
  feature_b_enabled = try(tobool(var.features["feature_b"]), true)
}

# Pattern 3: Handle null tag values
locals {
  raw_tags = {
    Name        = var.resource_name
    Owner       = var.owner        # might be null
    Project     = var.project      # might be null
    CostCenter  = var.cost_center  # might be null
  }

  # Filter out null values from tags map
  filtered_tags = {
    for key, value in local.raw_tags :
    key => value
    if value != null && value != ""
  }
}
```

## Summary

`try()` in OpenTofu provides null-safe attribute access and graceful fallbacks for optional configuration. Unlike using `lookup()` or `can()`, `try()` handles any type of expression failure, including type errors and null dereferences. Multiple fallback expressions are supported - `try(expr1, expr2, default_value)` - enabling cascading fallback logic for complex optional configurations. The key use case is accessing optional nested object attributes where the parent object might be null.
