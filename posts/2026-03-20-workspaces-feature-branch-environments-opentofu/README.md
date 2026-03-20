# How to Use Workspaces for Feature Branch Environments in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Workspaces, Feature Branch, Ephemeral Environments, CI/CD

Description: Learn how to use OpenTofu workspaces to create ephemeral feature branch environments for testing infrastructure changes in isolation before merging to main.

## Introduction

OpenTofu workspaces allow you to maintain multiple independent state files from the same configuration. Combined with CI/CD pipelines, workspaces enable ephemeral feature branch environments — isolated infrastructure instances created for each feature branch, tested, then destroyed when the branch is merged or closed. This pattern catches infrastructure issues before they reach staging or production.

## Workspace Basics for Feature Branches

```bash
# Create a workspace for a feature branch
git checkout -b feature/add-redis-cache
# Work on infrastructure changes...

# Create a matching workspace
tofu workspace new feature-add-redis-cache
tofu workspace show  # → feature-add-redis-cache

# Deploy to the feature workspace
tofu apply  # Creates isolated infrastructure for this branch

# When done: destroy and delete workspace
tofu destroy
tofu workspace select default
tofu workspace delete feature-add-redis-cache
```

## Workspace-Aware Configuration

```hcl
# main.tf - use workspace name to parameterize resources

locals {
  workspace    = terraform.workspace
  is_default   = terraform.workspace == "default"
  environment  = local.is_default ? "production" : local.workspace

  # Sizing based on workspace
  instance_type = local.is_default ? "t3.large" : "t3.small"
  db_instance   = local.is_default ? "db.t3.medium" : "db.t3.micro"
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.app.id
  instance_type = local.instance_type

  tags = {
    Name        = "${local.environment}-app"
    Environment = local.environment
    Workspace   = local.workspace
    Temporary   = local.is_default ? "false" : "true"
  }
}

resource "aws_db_instance" "main" {
  identifier     = "${local.environment}-db"
  instance_class = local.db_instance

  # Feature branch databases auto-delete on destroy
  skip_final_snapshot = !local.is_default
  deletion_protection = local.is_default
}
```

## Naming Convention for Feature Workspaces

```bash
# Sanitize branch name for workspace naming
branch_to_workspace() {
  local BRANCH="$1"
  # Replace slashes, special chars with hyphens
  echo "feature-$(echo "$BRANCH" | sed 's|[^a-zA-Z0-9]|-|g' | tr '[:upper:]' '[:lower:]' | sed 's/--*/-/g' | sed 's/^-//;s/-$//')"
}

# Examples:
branch_to_workspace "feature/add-redis-cache"  # → feature-feature-add-redis-cache
branch_to_workspace "fix/bug-123"              # → feature-fix-bug-123
branch_to_workspace "feat/user-auth-v2"        # → feature-feat-user-auth-v2
```

## GitHub Actions: Create Feature Environment on PR Open

```yaml
# .github/workflows/feature-env.yml
name: Feature Environment

on:
  pull_request:
    types: [opened, synchronize, reopened, closed]
    paths:
      - 'infrastructure/**'

env:
  WORKSPACE_NAME: feature-${{ github.event.pull_request.number }}

jobs:
  create-or-update:
    if: github.event.action != 'closed'
    runs-on: ubuntu-latest
    environment: feature-environments

    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_FEATURE }}
          aws-region: us-east-1

      - name: Create or select workspace
        working-directory: infrastructure
        run: |
          tofu init
          tofu workspace new "$WORKSPACE_NAME" 2>/dev/null || \
            tofu workspace select "$WORKSPACE_NAME"

      - name: Deploy feature environment
        working-directory: infrastructure
        run: tofu apply -auto-approve

      - name: Comment environment URL on PR
        uses: actions/github-script@v7
        with:
          script: |
            const outputs = require('child_process')
              .execSync('cd infrastructure && tofu output -json')
              .toString();
            const parsed = JSON.parse(outputs);
            const url = parsed.app_url?.value || 'Check outputs for URL';

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `## Feature Environment Ready\n**Environment:** \`${process.env.WORKSPACE_NAME}\`\n**URL:** ${url}\n\nThis environment will be destroyed when the PR is closed.`
            });

  destroy:
    if: github.event.action == 'closed'
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN_FEATURE }}
          aws-region: us-east-1

      - name: Destroy feature environment
        working-directory: infrastructure
        run: |
          tofu init
          tofu workspace select "$WORKSPACE_NAME" 2>/dev/null || exit 0
          tofu destroy -auto-approve
          tofu workspace select default
          tofu workspace delete "$WORKSPACE_NAME"
```

## Cost Controls for Feature Environments

```hcl
# variables.tf
variable "max_feature_env_age_days" {
  type    = number
  default = 7
  description = "Maximum age for feature environments in days"
}

# Add auto-shutdown tags for cloud cost management
resource "aws_instance" "app" {
  # ...
  tags = {
    AutoShutdown = terraform.workspace == "default" ? "false" : "true"
    MaxAgeDate   = timeadd(timestamp(), "${var.max_feature_env_age_days * 24}h")
    Workspace    = terraform.workspace
  }
}
```

```bash
#!/bin/bash
# cleanup-stale-feature-workspaces.sh - Run daily

# List all feature workspaces older than 7 days
tofu workspace list | grep "^  feature-" | while read -r WORKSPACE; do
  WORKSPACE=$(echo "$WORKSPACE" | tr -d ' ')

  # Get workspace creation date from Terraform Cloud API
  WORKSPACE_ID=$(get_workspace_id "$WORKSPACE")
  CREATED=$(curl -s \
    -H "Authorization: Bearer $TF_TOKEN" \
    "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID" | \
    jq -r '.data.attributes."created-at"')

  AGE_DAYS=$(( ($(date +%s) - $(date -d "$CREATED" +%s)) / 86400 ))

  if [ "$AGE_DAYS" -gt 7 ]; then
    echo "Destroying stale workspace $WORKSPACE (${AGE_DAYS} days old)"
    tofu workspace select "$WORKSPACE"
    tofu destroy -auto-approve
    tofu workspace select default
    tofu workspace delete "$WORKSPACE"
  fi
done
```

## Listing and Managing Feature Workspaces

```bash
# List all workspaces
tofu workspace list

# Output:
# * default
#   feature-123
#   feature-456
#   staging

# List via Terraform Cloud API with details
curl -s \
  -H "Authorization: Bearer $TF_TOKEN" \
  "https://app.terraform.io/api/v2/organizations/my-company/workspaces?search%5Bname%5D=feature-" | \
  jq '.data[] | {name: .attributes.name, created: .attributes."created-at", resources: .attributes."resource-count"}'
```

## Conclusion

Workspaces for feature branches enable ephemeral infrastructure testing with the same configuration as production, parameterized by workspace name. The GitHub Actions workflow creates the workspace when a PR opens, deploys the infrastructure, posts the URL as a PR comment, and destroys everything when the PR closes. Cost controls — reduced instance sizes for non-default workspaces, automatic shutdown tags, and stale workspace cleanup — prevent feature environments from accumulating costs. This pattern is most valuable for infrastructure-heavy applications where testing with real (though smaller) cloud resources catches issues that unit tests miss.
