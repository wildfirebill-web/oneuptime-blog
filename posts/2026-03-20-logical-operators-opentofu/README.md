# How to Use Logical Operators in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, HCL, Logical Operators, Expressions, Booleans, Infrastructure as Code, DevOps

Description: A guide to using logical operators in OpenTofu HCL expressions to combine boolean conditions for complex configuration logic.

## Introduction

Logical operators in OpenTofu combine boolean expressions to create complex conditions. They are fundamental to conditional resource creation, validation rules, and lifecycle conditions. OpenTofu supports AND (`&&`), OR (`||`), and NOT (`!`) operators.

## Logical Operators

```hcl
# &&  Logical AND (both must be true)

# ||  Logical OR (at least one must be true)
# !   Logical NOT (inverts boolean)

variable "environment" { type = string }
variable "region" { type = string }
variable "enable_ha" { type = bool default = true }

locals {
  is_prod      = var.environment == "prod"
  is_us_region = startswith(var.region, "us-")

  # AND: both conditions must be true
  prod_us = local.is_prod && local.is_us_region

  # OR: at least one condition must be true
  prod_or_ha = local.is_prod || var.enable_ha

  # NOT: invert boolean
  not_prod = !local.is_prod
}
```

## AND Operator in Conditionals

```hcl
variable "create_nat_gateway" {
  type    = bool
  default = true
}

# Create NAT gateway only when not in prod AND flag is set
resource "aws_nat_gateway" "main" {
  count = (!local.is_prod && var.create_nat_gateway) ? 1 : 0

  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
}

# Enable enhanced monitoring for prod AND large instances
resource "aws_db_instance" "main" {
  monitoring_interval = (local.is_prod && var.instance_class == "db.r5.large") ? 30 : 0
}
```

## OR Operator for Multiple Conditions

```hcl
variable "team" {
  type = string
}

locals {
  # Allow access if in prod OR if platform team
  requires_approval = local.is_prod || var.team == "platform"

  # Enable backup for prod OR staging
  enable_backup = var.environment == "prod" || var.environment == "staging"
}

resource "aws_backup_plan" "main" {
  count = local.enable_backup ? 1 : 0
  name  = "backup-plan-${var.environment}"

  rule {
    rule_name         = "daily"
    target_vault_name = aws_backup_vault.main.name
    schedule          = "cron(0 5 ? * * *)"
  }
}
```

## NOT Operator

```hcl
variable "use_existing_sg" {
  type    = bool
  default = false
}

# Create security group only when NOT using existing one
resource "aws_security_group" "app" {
  count  = !var.use_existing_sg ? 1 : 0
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
}

locals {
  # Invert flag
  create_new_sg = !var.use_existing_sg
}
```

## Complex Conditions with Multiple Operators

```hcl
variable "compliance_mode" {
  type    = bool
  default = false
}

variable "data_classification" {
  type    = string
  default = "internal"
}

locals {
  # Enable encryption when: compliance mode OR prod AND sensitive data
  enable_encryption = var.compliance_mode || (
    local.is_prod && (
      var.data_classification == "confidential" ||
      var.data_classification == "restricted"
    )
  )
}

resource "aws_s3_bucket_server_side_encryption_configuration" "app" {
  bucket = aws_s3_bucket.app.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = local.enable_encryption ? "aws:kms" : "AES256"
    }
  }
}
```

## Logical Operators in Validation

```hcl
variable "replica_count" {
  type = number
  validation {
    # Must be positive AND not too large
    condition     = var.replica_count > 0 && var.replica_count <= 10
    error_message = "Replica count must be between 1 and 10."
  }
}

variable "environment" {
  type = string
  validation {
    # Must be one of the allowed values
    condition     = var.environment == "dev" || var.environment == "staging" || var.environment == "prod"
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "region" {
  type = string
  validation {
    # Either US or EU regions
    condition     = startswith(var.region, "us-") || startswith(var.region, "eu-")
    error_message = "Only US and EU regions are supported."
  }
}
```

## Short-Circuit Evaluation

```hcl
variable "config_bucket" {
  type    = string
  default = null
}

locals {
  # Short-circuit: if config_bucket is null, second condition is not evaluated
  bucket_has_content = var.config_bucket != null && length(var.config_bucket) > 0

  # With OR: if first is true, second is not evaluated
  use_default_config = var.config_bucket == null || var.config_bucket == ""
}
```

## Logical Operators in Lifecycle Conditions

```hcl
resource "aws_db_instance" "main" {
  identifier        = "myapp-db"
  engine            = "postgres"
  instance_class    = var.db_instance_class

  lifecycle {
    precondition {
      # Require Multi-AZ AND deletion protection for production
      condition     = !local.is_prod || (var.multi_az == true && var.deletion_protection == true)
      error_message = "Production databases must have Multi-AZ and deletion protection enabled."
    }
  }
}
```

## Conclusion

Logical operators (`&&`, `||`, `!`) are essential for composing complex conditions from simpler boolean expressions. AND ensures all conditions must be true, OR requires at least one, and NOT inverts a boolean. Combine them with comparison operators to express sophisticated infrastructure policies: environment-specific configurations, compliance requirements, feature flags, and validation rules. When building complex conditions, use parentheses for clarity and consider extracting to locals for readability and reuse.
