# How to Use Variable Validation Rules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Validation

Description: Learn how to write variable validation rules in OpenTofu using the validation block to enforce constraints on input values before resource creation.

## Introduction

Variable validation in OpenTofu lets you define constraints on input variables that are checked before any resources are created. Unlike preconditions (which have access to all resource attributes), variable validation runs in isolation — only the variable being validated is accessible. Use it to enforce format requirements, value ranges, allowed lists, and other constraints on module inputs.

## Basic Validation Syntax

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "production"], var.environment)
    error_message = "Environment must be one of: dev, staging, production. Got: ${var.environment}"
  }
}
```

## Common Validation Patterns

### Non-Empty String

```hcl
variable "bucket_name" {
  type = string

  validation {
    condition     = length(var.bucket_name) > 0
    error_message = "Bucket name cannot be empty"
  }
}
```

### String Length Constraints

```hcl
variable "db_password" {
  type      = string
  sensitive = true

  validation {
    condition     = length(var.db_password) >= 16
    error_message = "Database password must be at least 16 characters long"
  }

  validation {
    condition     = length(var.db_password) <= 128
    error_message = "Database password must not exceed 128 characters"
  }
}
```

### Regex Pattern Matching

```hcl
variable "subdomain" {
  type = string

  validation {
    condition     = can(regex("^[a-z][a-z0-9-]{1,61}[a-z0-9]$", var.subdomain))
    error_message = "Subdomain must start with a letter, contain only lowercase letters, digits, hyphens, and be 3-63 chars"
  }
}

variable "aws_account_id" {
  type = string

  validation {
    condition     = can(regex("^[0-9]{12}$", var.aws_account_id))
    error_message = "AWS account ID must be exactly 12 digits"
  }
}
```

### Numeric Range

```hcl
variable "replica_count" {
  type = number

  validation {
    condition     = var.replica_count >= 1 && var.replica_count <= 10
    error_message = "Replica count must be between 1 and 10. Got: ${var.replica_count}"
  }
}
```

### CIDR Block Validation

```hcl
variable "vpc_cidr" {
  type = string

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid CIDR block. Got: ${var.vpc_cidr}"
  }
}
```

### List Validation

```hcl
variable "availability_zones" {
  type = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones are required for high availability"
  }

  validation {
    condition     = length(var.availability_zones) <= 3
    error_message = "No more than 3 availability zones are supported"
  }
}
```

### Cross-Field Validation (Multiple Variables)

Variable validation can only reference the variable being validated. For multi-variable checks, use local validation:

```hcl
# Single variable validation
variable "min_capacity" {
  type = number
  validation {
    condition     = var.min_capacity >= 1
    error_message = "min_capacity must be at least 1"
  }
}

variable "max_capacity" {
  type = number
  validation {
    condition     = var.max_capacity >= 1
    error_message = "max_capacity must be at least 1"
  }
}

# Cross-variable validation via locals + null_resource trick
# or via resource preconditions (which have access to all values)
```

## Multiple Validation Blocks

A variable can have multiple validation blocks, each checked independently:

```hcl
variable "instance_type" {
  type = string

  validation {
    condition     = can(regex("^t3\\.", var.instance_type)) || can(regex("^m5\\.", var.instance_type))
    error_message = "Instance type must be in the t3 or m5 family"
  }

  validation {
    condition     = !contains(["t3.nano", "t3.micro"], var.instance_type)
    error_message = "t3.nano and t3.micro are not supported in production deployments"
  }
}
```

## Validation Error Output

```bash
tofu apply

# Error: Invalid value for variable
#
#   on main.tf line 5:
#    5: variable "environment" {
#     ├────────────────
#     │ var.environment is "sandbox"
#
# Environment must be one of: dev, staging, production. Got: sandbox
```

## Conclusion

Variable validation is the first line of defense against misconfigured modules. Use it to enforce non-negotiable constraints on inputs: allowed values, format requirements, length limits, and numeric ranges. Write clear error messages that include both the requirement and the actual value. For constraints that require access to resource attributes or other variables, use preconditions instead — they run with full access to all resource values.
