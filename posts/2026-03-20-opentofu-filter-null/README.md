# How to Filter Null Values from Collections in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Null, Collections, Functions, Filtering

Description: Learn how to filter null values from lists and maps in OpenTofu using for expressions, compact, and try to build clean, null-safe configurations from optional inputs.

## Overview

Optional inputs, conditional resources, and data source lookups often produce `null` values in OpenTofu. Passing nulls into resource arguments or `for_each` causes errors. This post covers the standard patterns for removing nulls from lists and maps before they reach resource blocks.

## Filter Nulls from a List

```hcl
variable "additional_cidrs" {
  type    = list(string)
  default = []
  description = "Optional additional CIDR blocks"
}

locals {
  base_cidrs = ["10.0.0.0/16", "172.16.0.0/12"]

  # Mix of definite and potentially null values
  all_cidrs_with_nulls = concat(
    local.base_cidrs,
    var.additional_cidrs,
    # Conditional expression produces null when feature is disabled
    var.enable_vpn ? [var.vpn_cidr] : [null],
  )

  # Filter out nulls using a for expression with if
  all_cidrs = [
    for cidr in local.all_cidrs_with_nulls :
    cidr if cidr != null
  ]
}

# compact() is a shorthand for filtering nulls and empty strings from string lists

locals {
  optional_security_groups = compact([
    aws_security_group.base.id,
    var.enable_monitoring ? aws_security_group.monitoring.id : null,
    var.enable_vpn        ? aws_security_group.vpn.id        : null,
    var.enable_bastion    ? aws_security_group.bastion.id    : null,
  ])
}

resource "aws_instance" "app" {
  ami                    = data.aws_ami.latest.id
  instance_type          = "t3.medium"
  vpc_security_group_ids = local.optional_security_groups
}
```

## Filter Null Values from a Map

```hcl
variable "app_settings" {
  type = object({
    database_url     = string
    redis_url        = optional(string)  # Optional
    sentry_dsn       = optional(string)  # Optional
    feature_flags    = optional(string)  # Optional
  })
}

locals {
  # Build environment variables map, excluding nulls
  env_vars = {
    for key, value in {
      DATABASE_URL  = var.app_settings.database_url
      REDIS_URL     = var.app_settings.redis_url
      SENTRY_DSN    = var.app_settings.sentry_dsn
      FEATURE_FLAGS = var.app_settings.feature_flags
    } :
    key => value
    if value != null
  }
}

resource "aws_ecs_task_definition" "app" {
  family                   = "app"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = 256
  memory                   = 512

  container_definitions = jsonencode([{
    name        = "app"
    image       = "my-app:latest"
    environment = [
      for key, value in local.env_vars : {
        name  = key
        value = value
      }
    ]
  }])
}
```

## Null-Safe Tag Merging

```hcl
variable "cost_center" {
  type    = string
  default = null
  description = "Optional cost center for billing (omitted if not set)"
}

variable "team_name" {
  type    = string
  default = null
}

variable "on_call_contact" {
  type    = string
  default = null
}

locals {
  # Build tags map with only non-null values
  optional_tags = {
    for key, value in {
      CostCenter   = var.cost_center
      Team         = var.team_name
      OnCallContact = var.on_call_contact
    } :
    key => value
    if value != null
  }

  required_tags = {
    Environment = var.environment
    ManagedBy   = "OpenTofu"
    Project     = var.project_name
  }

  # Merge required + optional (no nulls in final map)
  all_tags = merge(local.required_tags, local.optional_tags)
}
```

## Filter null Resources from Conditional Creation

```hcl
# When some resources are conditionally created, their references may be null
variable "enable_waf" {
  type    = bool
  default = false
}

variable "enable_shield" {
  type    = bool
  default = false
}

resource "aws_wafv2_web_acl" "main" {
  count = var.enable_waf ? 1 : 0
  name  = "main-waf"
  scope = "REGIONAL"

  default_action { allow {} }
  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "main-waf"
    sampled_requests_enabled   = true
  }
}

resource "aws_shield_protection" "main" {
  count        = var.enable_shield ? 1 : 0
  name         = "alb-shield"
  resource_arn = aws_lb.main.arn
}

locals {
  # Collect protection ARNs, filtering nulls from disabled resources
  active_protections = compact([
    var.enable_waf    ? try(aws_wafv2_web_acl.main[0].arn, null)    : null,
    var.enable_shield ? try(aws_shield_protection.main[0].id, null) : null,
  ])
}

output "active_protection_count" {
  value = length(local.active_protections)
}
```

## Filter Stale Data from Data Source Results

```hcl
# Data sources return all items; filter to only usable ones
data "aws_instances" "all" {
  filter {
    name   = "tag:Environment"
    values = [var.environment]
  }
}

data "aws_instance" "details" {
  for_each    = toset(data.aws_instances.all.ids)
  instance_id = each.value
}

locals {
  # Filter to only running instances with a private IP assigned
  healthy_instances = {
    for id, inst in data.aws_instance.details :
    id => inst
    if inst.instance_state == "running" && inst.private_ip != null
  }

  # Extract IPs from healthy instances
  healthy_ips = values(local.healthy_instances)[*].private_ip
}

output "healthy_instance_count" {
  value = length(local.healthy_instances)
}

output "healthy_private_ips" {
  value = local.healthy_ips
}
```

## Summary

Filtering nulls from OpenTofu collections requires three techniques depending on the data type: `compact()` for simple string lists with nulls or empty strings, `for ... if value != null` for maps and typed lists, and `try(expr, null)` to safely convert potential errors into filterable nulls. Always filter before passing to `for_each`, `count`-indexed references, or `jsonencode` - all of which will error on null values.
