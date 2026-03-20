# How to Use Type Constraints in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HCL, Type Constraints, Variables, Type System, Infrastructure as Code, DevOps

Description: A guide to using type constraints in OpenTofu to enforce input types for variables and outputs, ensuring configuration correctness.

## Introduction

Type constraints in OpenTofu specify the expected type of a variable or output. When a value is provided that doesn't match the type constraint, OpenTofu attempts automatic conversion, and if that fails, it reports an error. Type constraints catch configuration mistakes early and make modules self-documenting.

## Primitive Types

```hcl
# string: text values

variable "environment" {
  type    = string
  default = "dev"
}

# number: integer or floating-point
variable "instance_count" {
  type    = number
  default = 3
}

# bool: true or false
variable "enable_logging" {
  type    = bool
  default = true
}
```

## Collection Types

```hcl
# list(type): ordered sequence of values of the same type
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# map(type): key-value pairs where all values have the same type
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Team        = "platform"
  }
}

# set(type): unordered collection of unique values
variable "allowed_ips" {
  type    = set(string)
  default = ["10.0.0.1", "10.0.0.2"]
}
```

## Structural Types

```hcl
# object: fixed set of named attributes with specific types
variable "database_config" {
  type = object({
    instance_class    = string
    allocated_storage = number
    multi_az          = bool
    backup_retention  = number
  })
  default = {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    multi_az          = false
    backup_retention  = 7
  }
}

# tuple: fixed-length sequence where each element has a specific type
variable "port_range" {
  type    = tuple([number, number])
  default = [8080, 8090]
}
```

## Complex Nested Types

```hcl
# List of objects
variable "security_groups" {
  type = list(object({
    name        = string
    description = string
    ports       = list(number)
  }))
}

# Map of objects
variable "environments" {
  type = map(object({
    instance_type = string
    min_capacity  = number
    max_capacity  = number
  }))
  default = {
    dev = {
      instance_type = "t3.micro"
      min_capacity  = 1
      max_capacity  = 3
    }
    prod = {
      instance_type = "t3.large"
      min_capacity  = 3
      max_capacity  = 20
    }
  }
}

# Using the complex type
resource "aws_autoscaling_group" "env" {
  for_each = var.environments

  min_size = each.value.min_capacity
  max_size = each.value.max_capacity
}
```

## Optional Object Attributes

```hcl
# optional() makes object attributes optional (OpenTofu 1.3+)
variable "server_config" {
  type = object({
    instance_type   = string
    ami_id          = string
    # Optional attributes with defaults
    key_name        = optional(string, null)
    user_data       = optional(string, "")
    instance_count  = optional(number, 1)
    enable_public_ip = optional(bool, false)
  })
}
```

## any Type

```hcl
# any: accepts any type (use sparingly)
variable "config_map" {
  type = any
  default = {}
}

# More specific is better:
# type = map(string)   -- all string values
# type = map(any)      -- mixed value types in map
```

## Type Conversion

```hcl
# OpenTofu automatically converts compatible types
variable "port" {
  type = number
}

# If user passes "8080" (string), it's auto-converted to 8080 (number)
# If conversion fails, OpenTofu reports an error

# Explicit conversion functions:
locals {
  port_as_string = tostring(var.port)
  count_as_number = tonumber("5")
  flag_as_bool = tobool("true")
  ids_as_list = tolist(toset(["a", "b", "a"]))  # Deduplicate
}
```

## Type Constraints in Outputs

```hcl
output "instance_ids" {
  value = aws_instance.web[*].id
  # Output type is inferred as list(string)
  # You can add explicit type in the description for documentation
  description = "List of EC2 instance IDs (list of strings)"
}

output "cluster_config" {
  value = {
    endpoint = aws_eks_cluster.main.endpoint
    name     = aws_eks_cluster.main.name
    version  = aws_eks_cluster.main.version
  }
  # Output type is inferred as object({endpoint: string, name: string, version: string})
}
```

## Type Constraints for Module Safety

```hcl
# modules/vpc/variables.tf
variable "cidr_block" {
  type        = string
  description = "VPC CIDR block"
  validation {
    # Validate format using type + validation
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "cidr_block must be a valid CIDR notation."
  }
}

variable "subnet_count" {
  type        = number
  description = "Number of subnets to create"
  validation {
    condition     = var.subnet_count > 0 && var.subnet_count <= 32
    error_message = "subnet_count must be between 1 and 32."
  }
}
```

## Conclusion

Type constraints in OpenTofu provide compile-time safety for variable inputs, making configurations more robust and self-documenting. Primitive types (`string`, `number`, `bool`) handle simple values, while collection types (`list`, `map`, `set`) handle groups of values, and structural types (`object`, `tuple`) handle structured data. Use `optional()` in object types to make attributes optional with defaults. Combine type constraints with `validation` blocks for complete input validation that catches both type errors and value range violations. The more specific your type constraints, the earlier OpenTofu can catch configuration mistakes.
