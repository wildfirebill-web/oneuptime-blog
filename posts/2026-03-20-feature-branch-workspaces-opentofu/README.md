# How to Use Feature Branch Infrastructure with OpenTofu Workspaces

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Workspaces, Feature Branches, CI/CD, Infrastructure as Code, Ephemeral Environments, DevOps

Description: Learn how to use OpenTofu workspaces to create isolated infrastructure environments for each feature branch, enabling independent testing without shared environment conflicts.

---

Feature branch infrastructure gives each developer or feature branch its own isolated cloud environment. Instead of everyone sharing a single staging environment, each branch gets its own ephemeral environment that's created when the branch is created and destroyed when it's merged. OpenTofu workspaces make this practical.

## Understanding OpenTofu Workspaces

Each workspace has its own state file, so `tofu apply` in the `feature-login` workspace creates resources completely separate from the `main` workspace.

```bash
# Create a workspace for a feature branch

tofu workspace new feature-user-auth

# Switch to an existing workspace
tofu workspace select feature-user-auth

# Show all workspaces
tofu workspace list

# Destroy and remove a workspace after merge
tofu destroy
tofu workspace select default
tofu workspace delete feature-user-auth
```

## Workspace-Aware Configuration

```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }

  backend "s3" {
    bucket = "my-tofu-state"
    key    = "feature-env/terraform.tfstate"  # Workspace name is appended automatically
    region = "us-east-1"
  }
}

provider "aws" {
  region = var.aws_region
}

locals {
  # Use workspace name as environment identifier
  environment = terraform.workspace
  is_default  = terraform.workspace == "default"

  # Size down resources for feature branches to save cost
  instance_count = local.is_default ? 2 : 1
  instance_type  = local.is_default ? "t3.medium" : "t3.micro"
}

# Create resources prefixed with workspace name to avoid naming conflicts
resource "aws_ecs_service" "app" {
  name          = "${terraform.workspace}-app"  # e.g., "feature-login-app"
  cluster       = var.ecs_cluster_arn
  desired_count = local.instance_count

  # Use branch-specific task definition
  task_definition = aws_ecs_task_definition.app.arn
}
```

## Workspace-Specific Variables

```hcl
# variables.tf
variable "workspace_configs" {
  description = "Configuration overrides per workspace type"
  type = map(object({
    instance_type  = string
    desired_count  = number
    retention_days = number
  }))

  default = {
    # Production workspace (main branch)
    default = {
      instance_type  = "t3.medium"
      desired_count  = 2
      retention_days = 90
    }
    # Feature branch workspaces - use smallest viable resources
    feature = {
      instance_type  = "t3.micro"
      desired_count  = 1
      retention_days = 7
    }
  }
}

locals {
  # Determine config based on workspace prefix
  workspace_type = startswith(terraform.workspace, "feature-") ? "feature" : "default"
  config         = var.workspace_configs[local.workspace_type]
}
```

## CI/CD Integration

```yaml
# .github/workflows/preview-environment.yml
name: Preview Environment
on:
  pull_request:
    types: [opened, synchronize, reopened, closed]

jobs:
  deploy-preview:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: us-east-1

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Create or switch workspace
        run: |
          WORKSPACE="pr-${{ github.event.pull_request.number }}"
          tofu workspace select $WORKSPACE || tofu workspace new $WORKSPACE
        working-directory: infrastructure

      - name: Deploy preview environment
        run: tofu apply -auto-approve -var="app_image=${{ env.IMAGE_TAG }}"
        working-directory: infrastructure

  destroy-preview:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest
    steps:
      - name: Destroy preview environment
        run: |
          WORKSPACE="pr-${{ github.event.pull_request.number }}"
          tofu workspace select $WORKSPACE
          tofu destroy -auto-approve
          tofu workspace select default
          tofu workspace delete $WORKSPACE
```

## Best Practices

- Use workspace names that match your branch naming convention - `feature-<ticket-number>` is a good pattern.
- Always size down resources in feature workspaces to reduce costs - full production capacity isn't needed for testing.
- Automate cleanup: destroy workspaces when PRs are merged or closed to avoid orphaned resources.
- Set cost budgets for workspace resources to prevent accidentally large deployments on feature branches.
- Use `terraform.workspace` to add the workspace name to all resource names and tags for easy identification and cleanup.
