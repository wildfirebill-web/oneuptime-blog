# How to Use Lookup Tables with Maps in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Maps, Lookup Tables, HCL, Infrastructure as Code

Description: Learn how to use maps as lookup tables in OpenTofu to simplify environment-specific configurations and eliminate repetitive conditional logic.

## Introduction

In OpenTofu, maps are key-value data structures that work exceptionally well as lookup tables. Instead of writing long chains of conditional expressions, you can define a map that maps an input key (like environment name) to a desired value, making your code cleaner and more maintainable.

## Basic Map Lookup

Define a map variable and look up a value by key:

```hcl
variable "environment" {
  type    = string
  default = "dev"
}

locals {
  instance_types = {
    dev     = "t3.micro"
    staging = "t3.medium"
    prod    = "m5.large"
  }

  selected_type = local.instance_types[var.environment]
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.selected_type
}
```

## Using lookup() for Safe Access with Defaults

The `lookup()` function retrieves a value from a map with a fallback default if the key doesn't exist:

```hcl
locals {
  replica_counts = {
    prod    = 3
    staging = 2
  }

  # Falls back to 1 for unknown environments
  replicas = lookup(local.replica_counts, var.environment, 1)
}
```

## Nested Maps for Multi-Dimensional Lookups

Use nested maps to handle multiple dimensions of configuration:

```hcl
locals {
  region_config = {
    "us-east-1" = {
      az_count = 3
      nat_gw   = true
      vpc_cidr = "10.0.0.0/16"
    }
    "eu-west-1" = {
      az_count = 2
      nat_gw   = true
      vpc_cidr = "10.1.0.0/16"
    }
    "ap-southeast-1" = {
      az_count = 2
      nat_gw   = false
      vpc_cidr = "10.2.0.0/16"
    }
  }

  current_region = local.region_config[var.aws_region]
}

resource "aws_vpc" "main" {
  cidr_block = local.current_region.vpc_cidr
}
```

## Maps as Feature Flags

Use maps to toggle feature flags per environment:

```hcl
locals {
  features = {
    dev = {
      enable_waf         = false
      enable_cloudfront  = false
      enable_backup      = false
    }
    prod = {
      enable_waf         = true
      enable_cloudfront  = true
      enable_backup      = true
    }
  }

  env_features = local.features[var.environment]
}

resource "aws_wafv2_web_acl_association" "app" {
  count        = local.env_features.enable_waf ? 1 : 0
  resource_arn = aws_lb.app.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

## Combining Maps with for_each

Iterate over a map to create multiple resources:

```hcl
variable "s3_buckets" {
  type = map(object({
    versioning = bool
    lifecycle  = number
  }))
  default = {
    "assets" = { versioning = true, lifecycle = 90 }
    "logs"   = { versioning = false, lifecycle = 30 }
    "backup" = { versioning = true, lifecycle = 365 }
  }
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.s3_buckets
  bucket   = "${var.project}-${each.key}-${var.environment}"
}

resource "aws_s3_bucket_versioning" "buckets" {
  for_each = { for k, v in var.s3_buckets : k => v if v.versioning }
  bucket   = aws_s3_bucket.buckets[each.key].id

  versioning_configuration {
    status = "Enabled"
  }
}
```

## Merging Maps

Combine base configuration with environment overrides:

```hcl
locals {
  base_tags = {
    Project   = var.project
    ManagedBy = "OpenTofu"
  }

  env_tags = {
    dev  = { CostCenter = "engineering", AutoShutdown = "true" }
    prod = { CostCenter = "production", AutoShutdown = "false" }
  }

  tags = merge(local.base_tags, local.env_tags[var.environment])
}
```

## Best Practices

- Use `locals` to define lookup maps close to where they are used.
- Prefer `lookup()` over direct indexing when the key may not exist.
- Document map keys in variable descriptions.
- Use `contains(keys(local.my_map), var.key)` to validate input keys early.
- Consider using `object` type constraints for nested maps to catch type errors at plan time.

## Conclusion

Maps as lookup tables in OpenTofu eliminate repetitive conditionals and make environment-specific configuration elegant and readable. Combined with `for_each` and `merge()`, they form the foundation of flexible, reusable OpenTofu modules.
