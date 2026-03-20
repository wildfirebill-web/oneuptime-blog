# How to Use can() and try() in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Functions

Description: Learn how to use can() and try() functions in OpenTofu for error handling and safe value evaluation.

`can()` and `try()` are error-handling functions that let you safely evaluate expressions that might fail. They're particularly useful when working with complex data structures, optional attributes, and type conversions.

## can()

Evaluates an expression and returns `true` if it succeeds without an error, or `false` if it fails:

```hcl
> can(1 / 0)
false

> can(tostring(42))
true

> can(jsondecode("{invalid}"))
false

> can(jsondecode("{\"key\": \"value\"}"))
true
```

### Validating CIDR Blocks

```hcl
variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "Must be a valid CIDR block (e.g., 10.0.0.0/16)."
  }
}
```

### Validating ARNs

```hcl
variable "role_arn" {
  type = string

  validation {
    condition     = can(regex("^arn:aws:iam::[0-9]{12}:role/.+$", var.role_arn))
    error_message = "Must be a valid IAM role ARN."
  }
}
```

## try()

Evaluates expressions in order and returns the first one that doesn't produce an error:

```hcl
> try(1 / 0, "fallback")
"fallback"

> try(tonumber("abc"), 0)
0

> try(tonumber("42"), 0)
42
```

### Safe Attribute Access

```hcl
variable "config" {
  type = any
  default = {
    database = {
      host = "db.example.com"
      port = 5432
    }
  }
}

locals {
  # Safe access with fallback
  db_port    = try(var.config.database.port, 5432)
  cache_host = try(var.config.cache.host, "localhost")
  debug_mode = try(var.config.app.debug, false)
}
```

### Handling Optional Values in Maps

```hcl
variable "environments" {
  type = map(object({
    instance_type = string
    min_size      = number
    max_size      = optional(number)
  }))
}

locals {
  # Try to get max_size, fall back to min_size * 3
  env_configs = {
    for name, env in var.environments :
    name => {
      instance_type = env.instance_type
      min_size      = env.min_size
      max_size      = try(env.max_size, env.min_size * 3)
    }
  }
}
```

### Safe JSON Parsing

```hcl
variable "metadata_json" {
  type    = string
  default = ""
}

locals {
  # Parse JSON if valid, otherwise use empty map
  metadata = try(jsondecode(var.metadata_json), {})
  
  # Safe nested access
  app_version = try(local.metadata.app.version, "unknown")
}
```

### Choosing Between Two Data Sources

```hcl
# Try to get an existing security group, fall back to a default

locals {
  sg_id = try(
    data.aws_security_group.existing.id,
    aws_security_group.default.id
  )
}
```

## can() vs try()

```hcl
# can() - returns boolean (use in conditions)
locals {
  is_valid_cidr = can(cidrnetmask(var.cidr))
}

# try() - returns the value or fallback (use when you need the value)
locals {
  safe_port = try(tonumber(var.port), 8080)
}

# Common pattern: validate with can(), use with try()
variable "port_string" {
  type = string
  
  validation {
    condition     = can(tonumber(var.port_string))
    error_message = "Port must be a valid number."
  }
}

locals {
  port = try(tonumber(var.port_string), 8080)
}
```

## Conclusion

`can()` and `try()` bring defensive programming to OpenTofu configurations. Use `can()` in validation blocks to check whether an expression is valid, and `try()` to safely access potentially absent data with a fallback. Together, they make your configurations resilient to optional or variable-format inputs from users, data sources, and external systems.
