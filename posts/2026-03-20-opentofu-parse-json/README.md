# How to Parse JSON Files for Configuration in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, JSON, jsondecode, Configuration, Data Files

Description: Learn how to parse JSON configuration files in OpenTofu using jsondecode and file() to drive infrastructure from external data sources.

## Overview

OpenTofu can read JSON configuration files using `file()` and `jsondecode()`, enabling data-driven infrastructure where resource definitions come from external configuration files rather than hardcoded HCL values.

## Step 1: Read and Parse JSON File

```hcl
# main.tf - Load configuration from JSON
locals {
  # Read and parse a JSON configuration file
  app_config = jsondecode(file("${path.module}/config/app-config.json"))
}

# config/app-config.json
# {
#   "environments": {
#     "production": {
#       "instance_type": "m5.xlarge",
#       "min_instances": 3,
#       "max_instances": 20,
#       "regions": ["us-east-1", "eu-west-1"]
#     },
#     "staging": {
#       "instance_type": "t3.large",
#       "min_instances": 1,
#       "max_instances": 5,
#       "regions": ["us-east-1"]
#     }
#   }
# }

resource "aws_autoscaling_group" "app" {
  name         = "app-${var.environment}"
  min_size     = local.app_config.environments[var.environment].min_instances
  max_size     = local.app_config.environments[var.environment].max_instances
  vpc_zone_identifier = module.vpc.private_subnets
}
```

## Step 2: Iterate Over JSON Array

```hcl
# config/users.json
# [
#   {"name": "alice", "role": "admin", "email": "alice@example.com"},
#   {"name": "bob", "role": "developer", "email": "bob@example.com"}
# ]

locals {
  users = jsondecode(file("${path.module}/config/users.json"))
}

resource "aws_iam_user" "from_json" {
  for_each = { for user in local.users : user.name => user }

  name = each.value.name
  path = "/users/"

  tags = {
    Email = each.value.email
    Role  = each.value.role
  }
}
```

## Step 3: Deep Nested JSON Access

```hcl
# Complex JSON with nested structures
locals {
  infrastructure = jsondecode(file("${path.module}/config/infrastructure.json"))

  # Access nested values safely
  db_config = try(
    local.infrastructure.databases[var.environment],
    local.infrastructure.databases["default"]
  )
}

# Use nested values
resource "aws_db_instance" "app" {
  instance_class    = local.db_config.instance_class
  allocated_storage = local.db_config.storage_gb
  multi_az          = local.db_config.multi_az
}
```

## Step 4: Merge JSON with Variables

```hcl
# Merge JSON defaults with variable overrides
locals {
  json_defaults = jsondecode(file("${path.module}/defaults.json"))

  # Override specific fields from variables
  merged_config = merge(local.json_defaults, {
    environment  = var.environment
    region       = var.region
    team         = var.team
  })

  # Add computed fields
  final_config = merge(local.merged_config, {
    deployment_id  = random_id.deployment.hex
    timestamp      = timestamp()
  })
}
```

## Summary

JSON configuration files parsed with `jsondecode(file(...))` enable infrastructure-as-data patterns where the same OpenTofu code deploys differently based on external JSON files. This separates the "what" (infrastructure code) from the "how" (configuration values), making it easy for non-Terraform users to modify configuration without understanding HCL. The `try()` function handles missing JSON keys gracefully, providing fallback values when a specific environment's configuration doesn't exist.
