# How to Define Input Variables in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variables, HCL, Infrastructure as Code, DevOps

Description: A comprehensive guide to defining and using input variables in OpenTofu to make configurations reusable and flexible.

## Introduction

Input variables in OpenTofu allow you to parameterize your configurations, making them reusable across different environments and contexts. Instead of hardcoding values, you declare variables and pass values at runtime or through variable files.

## Basic Variable Declaration

```hcl
# variables.tf

# Minimal variable (type is 'any' if not specified)
variable "aws_region" {}

# Variable with type constraint
variable "environment" {
  type = string
}

# Variable with description and default
variable "instance_type" {
  type        = string
  description = "The EC2 instance type to use"
  default     = "t3.micro"
}

# Variable with validation
variable "port" {
  type        = number
  description = "Application port number"
  default     = 8080

  validation {
    condition     = var.port >= 1024 && var.port <= 65535
    error_message = "Port must be between 1024 and 65535."
  }
}
```

## Variable Types

```hcl
# Primitive types
variable "name" {
  type = string
}

variable "count_value" {
  type = number
}

variable "enabled" {
  type = bool
}

# Collection types
variable "availability_zones" {
  type = list(string)
}

variable "instance_tags" {
  type = map(string)
}

variable "unique_ids" {
  type = set(string)
}

# Structural types
variable "network_config" {
  type = object({
    vpc_cidr    = string
    subnet_cidr = string
    public      = bool
  })
}

variable "server_configs" {
  type = list(object({
    name          = string
    instance_type = string
    count         = number
  }))
}
```

## Using Variables in Configurations

```hcl
# Reference a variable with var.<name>
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  count         = var.instance_count

  tags = {
    Environment = var.environment
    Name        = "${var.project_name}-web"
  }
}

# Variables can be used in expressions
locals {
  instance_name = "${var.project_name}-${var.environment}"
  is_production = var.environment == "prod"
}
```

## Complete variables.tf Example

```hcl
# variables.tf - Complete example for an AWS project

variable "aws_region" {
  type        = string
  description = "AWS region to deploy resources"
  default     = "us-east-1"
}

variable "environment" {
  type        = string
  description = "Deployment environment (dev, staging, prod)"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "project_name" {
  type        = string
  description = "Name of the project (used for resource naming)"
}

variable "instance_count" {
  type        = number
  description = "Number of web server instances to create"
  default     = 1
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"
}

variable "allowed_cidr_blocks" {
  type        = list(string)
  description = "CIDR blocks allowed to access the web servers"
  default     = ["0.0.0.0/0"]
}

variable "tags" {
  type        = map(string)
  description = "Additional tags to apply to all resources"
  default     = {}
}
```

## Sensitive Variables

```hcl
variable "database_password" {
  type        = string
  description = "Password for the database"
  sensitive   = true  # Masks the value in logs and plan output
}
```

## Conclusion

Input variables are the mechanism that transforms a hardcoded OpenTofu configuration into a reusable infrastructure template. By declaring variables with appropriate types, descriptions, defaults, and validation rules, you create self-documenting configurations that can be safely reused across environments. Always use the most specific type constraint possible to catch configuration errors early.
