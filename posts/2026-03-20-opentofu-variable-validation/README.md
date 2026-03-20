# How to Use Variable Validation Rules in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Validation

Description: Learn how to add validation rules to OpenTofu input variables to catch invalid values early, before any resources are created.

## Introduction

Variable validation blocks let you define conditions that input variable values must satisfy. If a value fails validation, OpenTofu reports an error immediately during plan — before reading any data sources or creating any resources. This provides the earliest possible feedback for incorrect inputs.

## Basic Validation

```hcl
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "environment must be one of: development, staging, production."
  }
}
```

## String Pattern Validation

```hcl
variable "bucket_name" {
  type        = string
  description = "S3 bucket name"

  validation {
    condition     = can(regex("^[a-z0-9-]+$", var.bucket_name))
    error_message = "Bucket name must contain only lowercase letters, numbers, and hyphens."
  }

  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be between 3 and 63 characters."
  }

  validation {
    condition     = !startswith(var.bucket_name, "-") && !endswith(var.bucket_name, "-")
    error_message = "Bucket name must not start or end with a hyphen."
  }
}
```

## Numeric Range Validation

```hcl
variable "instance_count" {
  type        = number
  description = "Number of EC2 instances to create"

  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 50
    error_message = "instance_count must be between 1 and 50. Got: ${var.instance_count}"
  }
}
```

## List Validation

```hcl
variable "availability_zones" {
  type        = list(string)
  description = "List of AZs for deployment"

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones are required for high availability."
  }

  validation {
    condition     = length(var.availability_zones) == length(toset(var.availability_zones))
    error_message = "availability_zones must not contain duplicates."
  }
}
```

## CIDR Block Validation

```hcl
variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"

  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "vpc_cidr must be a valid CIDR block. Got: ${var.vpc_cidr}"
  }

  validation {
    condition     = cidrcontains("10.0.0.0/8", var.vpc_cidr)
    error_message = "vpc_cidr must be within the private 10.0.0.0/8 address space."
  }
}
```

## Instance Type Validation

```hcl
variable "instance_type" {
  type        = string
  description = "EC2 instance type"

  validation {
    condition = contains([
      "t3.micro", "t3.small", "t3.medium",
      "t3.large", "t3.xlarge", "t3.2xlarge",
      "m5.large", "m5.xlarge", "m5.2xlarge"
    ], var.instance_type)
    error_message = "instance_type must be one of the approved instance types. Got: ${var.instance_type}"
  }
}
```

## Map Key Validation

```hcl
variable "tags" {
  type        = map(string)
  description = "Resource tags"

  validation {
    condition     = contains(keys(var.tags), "Environment")
    error_message = "tags must include an 'Environment' key."
  }

  validation {
    condition     = contains(keys(var.tags), "Owner")
    error_message = "tags must include an 'Owner' key."
  }
}
```

## Multiple Validations per Variable

```hcl
variable "database_name" {
  type = string

  validation {
    condition     = length(var.database_name) >= 1
    error_message = "Database name cannot be empty."
  }

  validation {
    condition     = length(var.database_name) <= 64
    error_message = "Database name must not exceed 64 characters."
  }

  validation {
    condition     = can(regex("^[a-zA-Z][a-zA-Z0-9_]*$", var.database_name))
    error_message = "Database name must start with a letter and contain only letters, numbers, and underscores."
  }
}
```

## Using can() for Safe Validation

```hcl
variable "ami_id" {
  type = string

  validation {
    # can() returns false if the expression errors (e.g., regex doesn't match)
    condition     = can(regex("^ami-[0-9a-f]+$", var.ami_id))
    error_message = "ami_id must be in the format 'ami-XXXXXXXX'. Got: ${var.ami_id}"
  }
}
```

## Validation Timing

Variable validation runs at the very beginning of `tofu plan`, before:
- Reading data sources
- Evaluating locals
- Planning resource changes

This makes validation the fastest feedback mechanism for incorrect inputs.

## Conclusion

Variable validation is the first line of defense against incorrect inputs. Define validation blocks with specific conditions and error messages that include the actual value received. Use `can()` with regex for format validation, `contains()` for enumerated values, and arithmetic comparisons for numeric ranges. Multiple validation blocks per variable allow clear, focused conditions with specific error messages for each type of constraint violation.
