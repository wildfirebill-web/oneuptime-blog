# How to Name Workspaces Following Best Practices in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Workspaces, Best Practices

Description: Learn workspace naming conventions and best practices for OpenTofu to create clear, consistent, and maintainable environment structures.

## Introduction

Workspace names become part of your state file paths, resource names, and tags. A consistent naming convention improves clarity, prevents mistakes, and makes your infrastructure easier to audit and manage. This guide covers proven naming patterns.

## Core Naming Principles

1. **Use lowercase**: Workspace names appear in resource names and URLs
2. **Use hyphens not underscores**: Better compatibility with resource naming
3. **Keep it short but descriptive**: 2-15 characters is ideal
4. **Make it environment-meaningful**: The name should clearly indicate the environment

## Environment-Based Naming (Most Common)

```bash
# Standard environment names

tofu workspace new development
tofu workspace new staging
tofu workspace new production

# Or abbreviated versions
tofu workspace new dev
tofu workspace new stg
tofu workspace new prod
```

Use the full name for clarity or abbreviations for brevity - just be consistent.

## Regional Workspace Naming

For multi-region deployments:

```bash
# Region-based
tofu workspace new us-east-1-production
tofu workspace new eu-west-1-production
tofu workspace new ap-southeast-1-production

# Or region code + environment
tofu workspace new use1-prod
tofu workspace new euw1-prod
tofu workspace new apse1-prod
```

## Account-Based Naming (Multi-Account)

For AWS multi-account setups:

```bash
# Account ID prefix
tofu workspace new 123456789012-production
tofu workspace new 234567890123-staging

# Account alias + environment
tofu workspace new prod-account-production
tofu workspace new dev-account-development
```

## Feature Branch Workspaces

For ephemeral, short-lived environments:

```bash
# Feature branch pattern
tofu workspace new feat-user-authentication
tofu workspace new feat-payment-api
tofu workspace new bugfix-timeout-issue

# PR number for traceability
tofu workspace new pr-1234
tofu workspace new pr-5678
```

## Team-Based Naming

For organizations where teams own environments:

```bash
# Team + environment
tofu workspace new platform-production
tofu workspace new data-team-staging
tofu workspace new commerce-development
```

## Using Workspace Names in Resources

Design your workspace names to produce clean resource names:

```hcl
locals {
  # Convert workspace name to a resource-name-safe format
  workspace = replace(terraform.workspace, ".", "-")
  resource_prefix = "${var.project_name}-${local.workspace}"
}

resource "aws_s3_bucket" "app" {
  # "myapp-production-data-123456789012"
  bucket = "${local.resource_prefix}-data-${data.aws_caller_identity.current.account_id}"
}
```

## Naming Convention Document

Create a `WORKSPACES.md` or add to your README:

```markdown
## Workspace Naming Convention

### Environment Workspaces
| Workspace | Purpose | Variables File |
|-----------|---------|----------------|
| development | Developer sandbox | development.tfvars |
| staging | Pre-production testing | staging.tfvars |
| production | Production | production.tfvars |

### Ephemeral Workspaces
| Pattern | Example | Purpose |
|---------|---------|---------|
| feat-{feature-name} | feat-auth-api | Feature development |
| pr-{pr-number} | pr-1234 | PR preview environments |

### Rules
1. All names must be lowercase
2. Use hyphens as separators
3. Maximum 30 characters
4. Ephemeral workspaces must be deleted within 7 days
```

## Validation in Configuration

Enforce your naming convention in code:

```hcl
# Validate workspace name format
check "workspace_name_convention" {
  assert {
    condition     = can(regex("^(development|staging|production|feat-[a-z0-9-]+|pr-[0-9]+)$", terraform.workspace))
    error_message = "Workspace name '${terraform.workspace}' does not follow the naming convention. Use: development, staging, production, feat-*, or pr-*."
  }
}
```

## Workspace Tagging

Use the workspace name in all resource tags for easy cost attribution and auditing:

```hcl
locals {
  common_tags = {
    Environment = terraform.workspace
    ManagedBy   = "OpenTofu"
    Project     = var.project_name
    Workspace   = terraform.workspace
  }
}

resource "aws_instance" "app" {
  # ...
  tags = merge(local.common_tags, {
    Name = "app-${terraform.workspace}"
  })
}
```

## Conclusion

A well-designed workspace naming convention makes your infrastructure self-documenting. Environment names like `production`, `staging`, and `development` are clear and widely understood. Ephemeral workspaces for feature branches should follow a predictable pattern that makes them easy to identify and clean up. Document your convention, enforce it through check blocks, and use workspace names consistently in resource naming and tagging for a clean, auditable infrastructure.
