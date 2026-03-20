# How to Use Boolean Variables in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variables, Booleans, HCL, Infrastructure as Code, DevOps

Description: A guide to using boolean type variables in OpenTofu to enable or disable features conditionally.

## Introduction

Boolean variables hold `true` or `false` values and are used for feature flags, conditional resource creation, and enabling/disabling configuration options. They are simple but powerful for controlling infrastructure behavior.

## Declaring Boolean Variables

```hcl
# Basic boolean variable

variable "enable_monitoring" {
  type = bool
}

# Boolean with default
variable "enable_deletion_protection" {
  type    = bool
  default = false
}

# Common boolean variables
variable "create_vpc" {
  type        = bool
  description = "Whether to create a new VPC or use existing"
  default     = true
}

variable "multi_az" {
  type        = bool
  description = "Enable Multi-AZ deployment for high availability"
  default     = false
}

variable "enable_enhanced_monitoring" {
  type        = bool
  description = "Enable enhanced monitoring metrics"
  default     = true
}
```

## Using Booleans for Conditional Resource Creation

```hcl
variable "create_bastion" {
  type    = bool
  default = false
}

# count = 1 creates the resource, count = 0 skips it
resource "aws_instance" "bastion" {
  count = var.create_bastion ? 1 : 0

  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "bastion-host"
  }
}

# Access conditional resource (returns list, index with [0])
output "bastion_ip" {
  value = var.create_bastion ? aws_instance.bastion[0].public_ip : null
}
```

## Boolean Feature Flags

```hcl
variable "enable_cdn" {
  type    = bool
  default = true
}

variable "enable_waf" {
  type    = bool
  default = false
}

variable "enable_access_logs" {
  type    = bool
  default = true
}

resource "aws_cloudfront_distribution" "cdn" {
  count = var.enable_cdn ? 1 : 0
  # ... CDN configuration
}

resource "aws_wafv2_web_acl_association" "cdn_waf" {
  count = var.enable_cdn && var.enable_waf ? 1 : 0
  # ... WAF association
}
```

## Environment-Based Boolean Patterns

```hcl
variable "environment" {
  type = string
}

locals {
  # Derive boolean features from environment
  is_production       = var.environment == "prod"
  enable_monitoring   = local.is_production
  enable_backup       = local.is_production || var.environment == "staging"
  enable_multi_az     = local.is_production
}

resource "aws_rds_cluster" "main" {
  # ...
  backup_retention_period = local.enable_backup ? 7 : 1
  deletion_protection     = local.is_production
}
```

## Boolean in tfvars Files

```hcl
# dev.tfvars
enable_monitoring         = false
enable_deletion_protection = false
multi_az                  = false
create_bastion            = true

# prod.tfvars
enable_monitoring         = true
enable_deletion_protection = true
multi_az                  = true
create_bastion            = false
```

```bash
# Apply with boolean overrides
tofu apply -var-file="prod.tfvars"
tofu apply -var="enable_monitoring=true" -var="multi_az=true"
```

## Boolean Type Conversion

```hcl
# Convert string to bool
locals {
  # tobool("true") = true, tobool("false") = false
  from_env = tobool(var.feature_flag_string)

  # Convert bool to string
  to_string = tostring(var.enable_monitoring)  # "true" or "false"

  # In expressions, 1 and 0 do NOT automatically convert to bool
  # Use explicit comparison:
  from_number = var.some_number != 0  # true if non-zero
}
```

## Conclusion

Boolean variables are the simplest and most direct way to implement feature flags and conditional configurations in OpenTofu. Using `count = var.feature_flag ? 1 : 0` is the idiomatic way to conditionally create resources. Combined with locals derived from environment variables, boolean flags make it easy to configure infrastructure differently for development, staging, and production with minimal duplication.
