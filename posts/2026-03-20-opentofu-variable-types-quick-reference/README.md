# How to Use the OpenTofu Variable Types Quick Reference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variable, Type, Quick Reference, Infrastructure as Code

Description: A quick reference for all OpenTofu variable types, type constraints, validation rules, and how to use variables effectively.

## Introduction

OpenTofu variables are typed and can include validation rules, descriptions, and defaults. This reference covers all variable types and patterns for effective variable usage.

## Primitive Types

```hcl
variable "region" {
  type    = string
  default = "us-east-1"
}

variable "instance_count" {
  type    = number
  default = 3
}

variable "enable_monitoring" {
  type    = bool
  default = true
}
```

## Collection Types

```hcl
# List: ordered, allows duplicates

variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Set: unordered, no duplicates (use for_each)
variable "allowed_environments" {
  type    = set(string)
  default = ["dev", "staging", "prod"]
}

# Map: key-value pairs with uniform value type
variable "instance_types" {
  type = map(string)
  default = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "m5.large"
  }
}
```

## Structural Types

```hcl
# Object: key-value pairs with named, typed fields
variable "database_config" {
  type = object({
    instance_class   = string
    allocated_storage = number
    multi_az         = bool
    engine_version   = optional(string, "15.4")  # with default
  })
}

# Tuple: ordered list with mixed types
variable "server_spec" {
  type = tuple([string, number, bool])
  # ["t3.medium", 50, true]
}

# List of objects
variable "ingress_rules" {
  type = list(object({
    from_port = number
    to_port   = number
    protocol  = string
    cidrs     = list(string)
  }))
}
```

## Variable Validation

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "environment must be dev, staging, or prod"
  }
}

variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^(t3|m5|c5)\\.", var.instance_type))
    error_message = "instance_type must start with t3., m5., or c5."
  }
}

variable "port" {
  type = number

  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "port must be between 1 and 65535"
  }
}
```

## Sensitive Variables

```hcl
variable "db_password" {
  type        = string
  sensitive   = true  # value masked in CLI output and logs
  description = "Database master password"
}
```

## Setting Variable Values

```hcl
# terraform.tfvars (auto-loaded)
environment = "prod"
region      = "us-east-1"

# environment.auto.tfvars (auto-loaded)
instance_count = 3
```

```bash
# Command line
tofu plan -var="environment=prod" -var="region=us-east-1"

# Variable file
tofu plan -var-file="prod.tfvars"

# Environment variable (TF_VAR_ prefix)
export TF_VAR_environment="prod"
export TF_VAR_db_password="secret"
tofu plan
```

## The any Type

```hcl
variable "tags" {
  type    = any  # accept any type
  default = {}   # often used for tags maps
}

# Better: use map(string) instead of any for tags
variable "tags" {
  type    = map(string)
  default = {}
}
```

## Nullable Variables

```hcl
variable "optional_config" {
  type     = string
  default  = null  # null means "not set"
  nullable = true  # explicit: value can be null
}

resource "aws_instance" "main" {
  # Use null to conditionally set values
  key_name = var.optional_config  # if null, no key pair assigned
}
```

## Summary

Use primitive types (`string`, `number`, `bool`) for simple values, collection types (`list`, `set`, `map`) for multiple values of the same type, and structural types (`object`, `tuple`) for complex configurations with named fields. Always add `description` and `validation` to public module variables. Mark sensitive variables as `sensitive = true` and use `optional()` in object types to provide per-field defaults.
