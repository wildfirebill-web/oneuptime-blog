# How to Name Workspaces Following Best Practices in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Workspaces

Description: Learn best practices for naming OpenTofu workspaces to keep environments organized, predictable, and safe to automate.

## Introduction

Workspace names become part of resource names, state file paths, and CI/CD automation. Choosing consistent, meaningful workspace names from the start reduces confusion, prevents naming collisions, and makes automation scripts more reliable.

## Core Naming Principles

Use lowercase letters, numbers, and hyphens only:

```bash
# Good names

tofu workspace new production
tofu workspace new staging
tofu workspace new development
tofu workspace new feature-user-auth

# Avoid
tofu workspace new Production    # uppercase causes inconsistency
tofu workspace new prod_env      # underscores less common than hyphens
tofu workspace new my workspace  # spaces not allowed
```

## Environment Tier Naming

Standard environment names are widely understood:

```bash
tofu workspace new development   # or: dev
tofu workspace new staging        # or: stage
tofu workspace new production     # or: prod
```

Pick one convention and stick to it. Mixing `prod` and `production` across teams creates confusion.

## Feature Branch Workspaces

Use branch-based names for ephemeral environments:

```bash
# Match workspace name to branch name
BRANCH="feature-payment-v2"
tofu workspace new "$BRANCH"

# Or sanitize the branch name for valid workspace naming
SAFE_BRANCH=$(echo "$BRANCH" | tr '/' '-' | tr '_' '-' | tr '[:upper:]' '[:lower:]')
tofu workspace new "$SAFE_BRANCH"
```

## Region-Scoped Names

For multi-region deployments within the same configuration:

```bash
tofu workspace new production-us-east-1
tofu workspace new production-eu-west-1
tofu workspace new staging-us-east-1
```

## Team or Tenant Scoping

For multi-tenant infrastructure:

```bash
tofu workspace new tenant-acme-corp
tofu workspace new tenant-globex
tofu workspace new tenant-initech
```

## Workspace Name Validation

```hcl
# Validate workspace names with a precondition
variable "allowed_workspaces" {
  type    = list(string)
  default = ["development", "staging", "production"]
}

resource "null_resource" "workspace_validation" {
  lifecycle {
    precondition {
      condition     = contains(var.allowed_workspaces, terraform.workspace)
      error_message = "Workspace '${terraform.workspace}' is not allowed. Use: ${join(", ", var.allowed_workspaces)}"
    }
  }
}
```

## Using Workspace Name in Resource Naming

```hcl
# Include workspace name in resources for easy identification
resource "aws_s3_bucket" "data" {
  # Workspace name appears in all resource names
  bucket = "acme-data-${terraform.workspace}-${var.region}"
}

resource "aws_eks_cluster" "main" {
  name = "acme-${terraform.workspace}-eks"
}
```

## Automation Patterns

```bash
# CI/CD: derive workspace from branch or environment variable
WORKSPACE="${ENVIRONMENT:-development}"

# Sanitize workspace name from any input
sanitize_workspace() {
  echo "$1" | tr '[:upper:]' '[:lower:]' | tr '/_.' '-' | sed 's/[^a-z0-9-]//g'
}

WORKSPACE=$(sanitize_workspace "$CI_BRANCH")
tofu workspace select -or-create "$WORKSPACE"
```

## What to Avoid

- Long names with special characters: `my_production_environment_2024` - hard to type and prone to errors
- Generic names: `test`, `temp`, `new` - unclear purpose, hard to clean up
- Numbers only: `1`, `2`, `3` - no semantic meaning
- Names that are too similar: `prod` and `production` in the same project - confusing

## Conclusion

Workspace names should be lowercase, hyphen-separated, and clearly communicate the environment's purpose. Follow a consistent naming convention across all projects (e.g., always use `production` never `prod`). For feature branch workspaces, derive the name from the branch and sanitize it to remove invalid characters. Include workspace names in all resource names and tags for easy identification in cloud consoles.
