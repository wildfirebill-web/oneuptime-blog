# How to Chain Conditional Logic in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Conditionals, HCL, Logic, Advanced

Description: Learn how to chain multiple conditional expressions in OpenTofu using nested ternaries, locals composition, and helper variables for complex multi-way logic.

## Introduction

Real-world infrastructure configurations often need multi-way conditional logic — not just true/false, but choices among three, four, or more options. OpenTofu's ternary expressions can be chained, and decomposing complex logic into named `locals` keeps it readable.

## Chained Ternary Expressions

For simple multi-way selection, chain ternary operators. Use parentheses and newlines for readability.

```hcl
variable "tier" {
  type = string  # "free", "starter", "professional", "enterprise"
}

locals {
  # Multi-way selection using chained conditionals
  instance_type = (
    var.tier == "free"         ? "t3.micro"  :
    var.tier == "starter"      ? "t3.small"  :
    var.tier == "professional" ? "t3.large"  :
    "m5.xlarge"  # enterprise - default/fallback
  )

  storage_gb = (
    var.tier == "free"         ? 10   :
    var.tier == "starter"      ? 50   :
    var.tier == "professional" ? 200  :
    1000  # enterprise
  )

  max_connections = (
    var.tier == "free"         ? 5    :
    var.tier == "starter"      ? 25   :
    var.tier == "professional" ? 100  :
    1000
  )
}
```

## Composing Conditionals Through Locals

Break complex logic into named intermediate values for clarity.

```hcl
variable "environment"       { type = string }
variable "region"            { type = string }
variable "compliance_mode"   { type = bool; default = false }
variable "high_availability" { type = bool; default = null }

locals {
  # Step 1: Derive basic flags
  is_prod     = var.environment == "prod"
  is_us       = startswith(var.region, "us-")
  requires_ha = var.high_availability != null ? var.high_availability : local.is_prod

  # Step 2: Derive configuration from flags
  needs_cmk = local.is_prod || var.compliance_mode
  needs_vpc = local.is_prod || local.is_us

  # Step 3: Select specific values from composed conditions
  db_instance_class = (
    local.is_prod && var.compliance_mode ? "db.r6g.xlarge" :
    local.is_prod                        ? "db.r6g.large"  :
    local.requires_ha                    ? "db.t3.small"   :
    "db.t3.micro"
  )

  backup_retention = (
    var.compliance_mode ? 365 :
    local.is_prod       ? 30  :
    7
  )
}
```

## Conditional Logic with can() and try()

Use `can()` to test whether an expression would succeed, enabling safe fallbacks.

```hcl
variable "vpc_config" {
  type = object({
    id             = string
    subnet_ids     = list(string)
    security_groups = list(string)
  })
  default = null
}

locals {
  # Check if vpc_config is provided and has valid subnet IDs
  has_valid_vpc = can(var.vpc_config.subnet_ids[0])

  # Use VPC config if available, otherwise compute defaults
  effective_subnet_ids = local.has_valid_vpc ? (
    var.vpc_config.subnet_ids
  ) : (
    data.aws_subnets.default.ids
  )
}
```

## Building Feature Flags from Multiple Conditions

```hcl
variable "environment"   { type = string }
variable "region"        { type = string }
variable "explicit_features" {
  type    = map(bool)
  default = {}
}

locals {
  # Derive implicit features from environment/region
  implicit_features = {
    enable_backup         = var.environment == "prod"
    enable_monitoring     = contains(["staging", "prod"], var.environment)
    enable_cdn            = var.environment == "prod" && local.is_us
    enable_multi_az       = var.environment == "prod"
    enable_read_replicas  = var.environment == "prod"
  }

  # Explicit features override implicit ones
  features = merge(local.implicit_features, var.explicit_features)
}

# Use feature flags to drive resource creation
resource "aws_db_instance" "read_replica" {
  count               = local.features.enable_read_replicas ? 1 : 0
  replicate_source_db = aws_db_instance.main.id
  instance_class      = "db.r6g.large"
}
```

## Conclusion

Chain conditionals by decomposing them into named `locals` that build on each other. This approach is more readable than deeply nested ternaries and makes the logic easy to unit test. Use `can()` and `try()` when accessing potentially null or undefined values to make your chains robust against missing inputs.
