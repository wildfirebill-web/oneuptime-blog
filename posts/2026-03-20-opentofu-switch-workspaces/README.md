# How to Switch Between Workspaces in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Workspaces

Description: Learn how to switch between OpenTofu workspaces using tofu workspace select to operate on different environment states from the same configuration.

## Introduction

The `tofu workspace select` command switches the active workspace. After switching, all subsequent `tofu plan`, `tofu apply`, and other commands operate on the state associated with the selected workspace. This allows the same configuration directory to manage multiple environments.

## Basic Switch

```bash
# Switch to the production workspace
tofu workspace select production

# Output:
# Switched to workspace "production".
```

```bash
# Confirm the switch
tofu workspace list
#   default
#   staging
# * production
```

## Create and Switch in One Step

OpenTofu v1.x supports creating a workspace if it doesn't exist:

```bash
# -or-create flag creates the workspace if it doesn't exist
tofu workspace select -or-create staging
```

This eliminates the need to check existence before selecting.

## Switching in CI/CD

```bash
# Deploy to the correct environment based on branch
if [ "$BRANCH" = "main" ]; then
  WORKSPACE="production"
else
  WORKSPACE="staging"
fi

tofu workspace select "$WORKSPACE"
tofu plan
tofu apply -auto-approve
```

## What Changes After Switching

After selecting a different workspace:

```bash
tofu workspace select staging

# tofu state list now shows staging resources
tofu state list

# tofu plan operates on staging state
tofu plan
```

The configuration files (`*.tf`) are shared — only the state changes.

## Workspace-Aware Resource Naming

```hcl
# Resources are named differently per workspace
resource "aws_s3_bucket" "data" {
  bucket = "acme-data-${terraform.workspace}"
}

# staging workspace → acme-data-staging
# production workspace → acme-data-production
```

After switching workspaces, `tofu plan` shows the resources for that environment.

## Switching with Different Variable Files

```bash
# Switch workspace and use environment-specific variables
tofu workspace select production
tofu apply -var-file=environments/production.tfvars

tofu workspace select staging
tofu apply -var-file=environments/staging.tfvars
```

## Confirming the Current Workspace

```bash
# Show only the current workspace name
tofu workspace show
# production
```

Use `tofu workspace show` (not `list`) when you only need the current workspace name in scripts.

## Safe Workspace Switching Pattern

```bash
# Always confirm workspace before destructive operations
CURRENT=$(tofu workspace show)
echo "Operating on workspace: $CURRENT"
read -p "Proceed? (yes/no): " CONFIRM
if [ "$CONFIRM" = "yes" ]; then
  tofu apply -auto-approve
fi
```

## Conclusion

Use `tofu workspace select <name>` to switch environments. The `-or-create` flag simplifies CI/CD scripts by handling both creation and selection in one command. Always confirm the active workspace with `tofu workspace show` before running destructive operations. Pair workspace selection with environment-specific variable files for complete environment isolation.
