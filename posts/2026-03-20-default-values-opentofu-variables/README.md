# How to Set Default Values for OpenTofu Variables

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variables, Defaults, Infrastructure as Code, DevOps

Description: A guide to setting and using default values for OpenTofu input variables to simplify configuration management.

## Introduction

Default values for OpenTofu variables make configurations easier to use by providing sensible presets that can be overridden when needed. A variable with a default is optional - users only need to provide a value if they want to override the default.

## Setting Default Values

```hcl
# String default

variable "aws_region" {
  type    = string
  default = "us-east-1"
}

# Number default
variable "instance_count" {
  type    = number
  default = 2
}

# Boolean default
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List default
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map default
variable "default_tags" {
  type = map(string)
  default = {
    ManagedBy = "OpenTofu"
    Project   = "default"
  }
}

# Object default
variable "server_config" {
  type = object({
    instance_type = string
    disk_size     = number
    monitoring    = bool
  })
  default = {
    instance_type = "t3.micro"
    disk_size     = 20
    monitoring    = false
  }
}
```

## Required vs Optional Variables

```hcl
# REQUIRED: No default provided
variable "project_name" {
  type        = string
  description = "Project name (required, must be provided)"
  # No default = required
}

# OPTIONAL: Default provided
variable "instance_type" {
  type        = string
  description = "EC2 instance type"
  default     = "t3.micro"  # Optional, has default
}
```

## Null as a Default Value

```hcl
# Use null when the variable should be optional and omitted if not set
variable "custom_ami" {
  type        = string
  description = "Custom AMI ID (uses latest if not specified)"
  default     = null  # null means 'not specified'
}

# Use in resource with conditional
resource "aws_instance" "web" {
  # Use custom AMI if provided, otherwise use latest
  ami = var.custom_ami != null ? var.custom_ami : data.aws_ami.latest.id
}
```

## Environment-Specific Defaults

```hcl
# Use locals to compute environment-based defaults
variable "environment" {
  type    = string
  default = "dev"
}

variable "instance_type" {
  type    = string
  default = null  # Will be set via local if null
}

locals {
  # Compute defaults based on environment
  instance_type_defaults = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }

  # Use explicit variable or fall back to environment default
  effective_instance_type = var.instance_type != null ? var.instance_type : local.instance_type_defaults[var.environment]
}

resource "aws_instance" "web" {
  instance_type = local.effective_instance_type
}
```

## Default Values in Practice

```bash
# Using default values (no -var flags needed)
tofu apply

# Override specific defaults
tofu apply -var="instance_count=5"

# Override multiple defaults
tofu apply \
  -var="environment=staging" \
  -var="instance_count=3" \
  -var="instance_type=t3.small"

# Override with variable file
tofu apply -var-file="prod.tfvars"
```

## Validating Defaults

```hcl
variable "environment" {
  type    = string
  default = "dev"  # Default must pass validation

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Must be dev, staging, or prod."
  }
}

# Note: the default value is also validated against the validation block
# So defaults must satisfy their own validation rules
```

## Conclusion

Default values in OpenTofu variables provide sensible presets that reduce the burden on users while still allowing customization. A well-designed set of defaults means you can apply a configuration with minimal input in development, while still supporting full customization in production. Always provide defaults for optional configuration and require explicit values for critical settings like environment names and project identifiers.
