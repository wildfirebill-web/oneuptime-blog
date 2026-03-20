# How to Build Dynamic Tags Maps with merge in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Merge, Tags, Resource Tagging, Best Practices

Description: Learn how to use OpenTofu's merge function to build dynamic tag maps that combine common organizational tags with resource-specific tags for consistent cloud resource tagging.

## Overview

`merge()` combines multiple maps into one, with later maps taking precedence for duplicate keys. This enables a layered tagging strategy: organization-wide defaults + environment tags + resource-specific tags.

## Step 1: Layered Tagging Strategy

```hcl
# main.tf - Layered tag system

locals {
  # Organization-wide mandatory tags
  org_tags = {
    Organization = "MyCompany"
    ManagedBy    = "OpenTofu"
    Repository   = "github.com/mycompany/infrastructure"
  }

  # Environment-level tags
  env_tags = {
    Environment = var.environment
    CostCenter  = lookup(var.cost_centers, var.environment, "unknown")
    Owner       = var.team_name
  }

  # Merge: resource-specific tags override environment, which overrides org
  common_tags = merge(local.org_tags, local.env_tags)
}

# Per-resource tags override common tags
resource "aws_instance" "web" {
  ami           = data.aws_ami.app.id
  instance_type = "m5.large"

  tags = merge(local.common_tags, {
    Name        = "web-server-${var.environment}"
    Role        = "web"
    AutoShutdown = var.environment != "production" ? "true" : "false"
  })
}
```

## Step 2: Dynamic Cost Allocation Tags

```hcl
# Build cost allocation tags from multiple sources
variable "project_metadata" {
  type = object({
    project_id  = string
    team        = string
    cost_center = string
    jira_ticket = optional(string)
  })
}

locals {
  cost_tags = merge(
    {
      "billing:project-id"  = var.project_metadata.project_id
      "billing:team"        = var.project_metadata.team
      "billing:cost-center" = var.project_metadata.cost_center
    },
    var.project_metadata.jira_ticket != null ? {
      "billing:jira" = var.project_metadata.jira_ticket
    } : {}
  )

  all_tags = merge(local.common_tags, local.cost_tags)
}
```

## Step 3: Conditional Tags

```hcl
# Add tags conditionally based on environment
locals {
  # Add Data Classification tag for production
  security_tags = merge(
    { "security:classification" = "internal" },
    var.environment == "production" ? {
      "security:data-sensitivity" = "high"
      "security:pci-in-scope"     = "false"
      "security:hipaa-in-scope"   = "false"
    } : {}
  )

  # Add backup tags for stateful resources
  backup_tags = var.enable_backup ? {
    "backup:enabled"   = "true"
    "backup:frequency" = "daily"
    "backup:retention" = "30"
  } : {
    "backup:enabled" = "false"
  }
}

resource "aws_db_instance" "app" {
  tags = merge(local.common_tags, local.security_tags, local.backup_tags, {
    Name = "app-database"
  })
}
```

## Step 4: Tag Inheritance Module

```hcl
# Module that accepts extra tags and merges with defaults
variable "tags" {
  type        = map(string)
  default     = {}
  description = "Additional tags to add to all resources"
}

locals {
  module_default_tags = {
    Module  = "database"
    Version = "1.0.0"
  }

  # Caller-provided tags override module defaults
  effective_tags = merge(local.module_default_tags, var.tags)
}

resource "aws_rds_cluster" "app" {
  # ...
  tags = local.effective_tags
}
```

## Summary

`merge()` in OpenTofu enables a tagging hierarchy where later arguments override earlier ones for duplicate keys. The pattern of `merge(org_tags, env_tags, resource_specific_tags)` ensures all resources inherit mandatory organization tags while allowing customization at each level. Conditional tags using ternary expressions with empty maps (`condition ? { key = value } : {}`) cleanly add optional tags without creating complex if/else chains. Store the `org_tags` and `env_tags` in a shared module or locals block and reference them across all resource files.
