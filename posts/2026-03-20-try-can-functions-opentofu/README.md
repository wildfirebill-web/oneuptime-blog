# How to Use try() and can() Functions in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Try, Can, Error Handling, HCL, Functions, Infrastructure as Code

Description: Learn how to use try() and can() in OpenTofu to safely access optional attributes, handle missing keys, and provide fallback values without causing configuration errors.

## Introduction

`try()` returns the first expression that doesn't produce an error. `can()` returns true if an expression evaluates without error. Together, they enable safe access to optional attributes, nullable values, and dynamically-typed data without causing `tofu plan` to fail.

## try() Function

```hcl
# try(expression1, expression2, ...) returns first non-error result

# If all expressions error, try() itself errors

variable "config" {
  default = {
    timeout = 30
    # retries is not set
  }
}

locals {
  # Safe access to optional attribute - falls back to default
  retries = try(var.config.retries, 3)
  # var.config.retries would error (key not found), so returns 3

  timeout = try(var.config.timeout, 60)
  # var.config.timeout = 30, so returns 30

  # Chain multiple fallbacks
  cache_ttl = try(var.config.cache.ttl, var.defaults.cache_ttl, 300)
}
```

## can() Function

```hcl
# can(expression) returns true if expression evaluates without error

variable "instance_config" {
  default = {
    instance_type = "t3.medium"
    # spot_price not set
  }
}

locals {
  # Check if optional field exists
  is_spot = can(var.instance_config.spot_price)
  # false - spot_price not in the map

  # Check if a conversion would succeed
  is_valid_number = can(tonumber("42"))   # true
  is_invalid      = can(tonumber("abc"))  # false
}
```

## Optional Resource Configuration

```hcl
variable "monitoring_config" {
  type    = any
  default = null   # Monitoring is optional
}

locals {
  # Only enable monitoring if config is provided
  enable_monitoring = var.monitoring_config != null

  # Safely access nested optional values
  monitoring_interval = try(var.monitoring_config.interval, 60)
  monitoring_endpoint = try(var.monitoring_config.endpoint, "")
}

resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = local.enable_monitoring ? 1 : 0

  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  threshold           = try(var.monitoring_config.cpu_threshold, 80)
  period              = local.monitoring_interval
  # ...
}
```

## Handling Optional Module Outputs

When consuming modules that may or may not produce certain outputs:

```hcl
# Module that conditionally creates resources
module "bastion" {
  count  = var.enable_bastion ? 1 : 0
  source = "./modules/bastion"
  # ...
}

locals {
  # Safely get bastion IP - null if bastion not created
  bastion_ip = try(module.bastion[0].public_ip, null)
}

# Use in security group rule
resource "aws_security_group_rule" "bastion_ssh" {
  count = local.bastion_ip != null ? 1 : 0

  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["${local.bastion_ip}/32"]
  security_group_id = aws_security_group.app.id
}
```

## Type-Safe Attribute Access

```hcl
variable "tags" {
  type    = any  # Flexible type - may be map or null
  default = null
}

locals {
  # Safe tag access - handle both map and null cases
  environment_tag = try(var.tags.Environment, "unknown")
  all_tags = try(var.tags, {})  # Default to empty map if null
}
```

## can() for Input Validation

```hcl
variable "cidr_block" {
  type = string
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "The cidr_block must be a valid CIDR notation (e.g., 10.0.0.0/16)."
  }
}

variable "instance_type" {
  type = string
  validation {
    condition = can(regex("^(t|m|c|r|i|d|h|x|z)[0-9][a-z]?\\.", var.instance_type))
    error_message = "Must be a valid AWS instance type (e.g., t3.medium)."
  }
}
```

## Difference: try() vs coalesce()

```hcl
# coalesce() returns first non-null, non-empty string
locals {
  value1 = coalesce(null, "", "fallback")  # "fallback" (skips null and "")
  value2 = coalesce("first", "second")     # "first"

  # try() catches errors - coalesce() does not
  safe_access = try(var.config.nested.deep.value, "default")  # Works
  # coalesce(var.config.nested.deep.value, "default") would ERROR if nested/deep don't exist
}
```

## Practical: Processing External Data

```hcl
data "http" "config" {
  url = "https://config.example.com/app.json"
}

locals {
  # Parse JSON safely
  app_config = try(jsondecode(data.http.config.response_body), {})

  # Access nested values safely
  db_host = try(local.app_config.database.host, "localhost")
  db_port = try(tonumber(local.app_config.database.port), 5432)
  db_name = try(local.app_config.database.name, "app")
}
```

## Conclusion

`try()` and `can()` provide error-safe expression evaluation for optional attributes, nullable values, and type conversions. Use `try(expr, default)` as a two-argument pattern for safe attribute access with a fallback. Use `can(expr)` in validation blocks to check if a conversion or attribute access would succeed. Avoid overusing them - if a variable should always have a value, require it explicitly. Use `try()` and `can()` specifically for genuinely optional configuration that has reasonable defaults.
