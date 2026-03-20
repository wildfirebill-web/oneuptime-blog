# How to Use Remote Execution with Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Remote Execution, Terraform Cloud, CI/CD

Description: Learn how to use remote execution with the OpenTofu cloud backend to run plans and applies in Terraform Cloud's managed infrastructure with streamed output to your terminal.

## Introduction

Remote execution runs `tofu plan` and `tofu apply` on Terraform Cloud's infrastructure rather than your local machine. Your configuration files are uploaded, the plan executes on a managed runner with cloud credentials injected via workspace variables, and output streams back to your terminal in real time. This provides consistent execution environments and centralized credential management.

## Configuring Remote Execution

```hcl
# main.tf - remote execution is the default for cloud backend

terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-infrastructure"
    }
  }
}
```

```bash
# Terraform Cloud workspace must have execution mode: Remote

# Set in workspace UI: Settings → General → Execution Mode → Remote

# Or via API:
curl -X PATCH \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID" \
  -d '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "execution-mode": "remote"
      }
    }
  }'
```

## Running Plans Remotely

```bash
# Standard plan - uploads config and runs in Terraform Cloud
tofu plan

# Output shows remote execution:
# Running plan in Terraform Cloud. Output will stream here.
# Waiting for the plan to start...
# Terraform v1.7.0
# on linux_amd64
#
# Initializing plugins and modules...
# ...
# Plan: 3 to add, 1 to change, 0 to destroy.

# Save plan output for apply (remote plans can be applied from UI or CLI)
tofu plan -out=plan.tfplan  # Note: tfplan files from remote runs are pointers, not local files
```

## Running Applies Remotely

```bash
# Apply with approval in Terraform Cloud (default)
tofu apply
# Terraform Cloud runs the plan, shows output, waits for approval in UI or CLI

# Auto-approve (use with caution, bypasses UI approval)
tofu apply -auto-approve

# Apply a saved plan
tofu apply plan.tfplan
```

## Workspace Variables for Remote Execution

```bash
# Set cloud credentials as workspace variables (injected during remote execution)
# No need to have AWS credentials locally

# Set via API
set_workspace_var() {
  local KEY="$1"
  local VALUE="$2"
  local CATEGORY="${3:-env}"
  local SENSITIVE="${4:-true}"

  curl -s -X POST \
    -H "Authorization: Bearer $TF_TOKEN" \
    -H "Content-Type: application/vnd.api+json" \
    "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID/vars" \
    -d "{
      \"data\": {
        \"type\": \"vars\",
        \"attributes\": {
          \"key\": \"$KEY\",
          \"value\": \"$VALUE\",
          \"category\": \"$CATEGORY\",
          \"sensitive\": $SENSITIVE
        }
      }
    }"
}

# Set AWS credentials in the workspace
set_workspace_var "AWS_ACCESS_KEY_ID" "$AWS_ACCESS_KEY_ID" "env" true
set_workspace_var "AWS_SECRET_ACCESS_KEY" "$AWS_SECRET_ACCESS_KEY" "env" true
set_workspace_var "AWS_DEFAULT_REGION" "us-east-1" "env" false
```

## Streaming Output

```bash
# Remote execution streams output in real time
tofu plan

# Typical output stream:
# Running plan in Terraform Cloud. Output will stream here. Waiting for the plan to start...
#
# Terraform v1.7.0
# on linux_amd64
# Preparing the remote plan...
# Counting objects: 8, done.
#
# Terraform used the selected providers to generate the following execution plan.
# ...
# Plan: 2 to add, 0 to change, 0 to destroy.

# View run in browser
# After running: a URL to the run in Terraform Cloud is printed
# https://app.terraform.io/app/my-company/workspaces/production/runs/run-abc123
```

## Run Triggers and Auto-Apply

```bash
# Configure workspace for automatic apply after successful plan
curl -X PATCH \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID" \
  -d '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "auto-apply": true
      }
    }
  }'
```

## CI/CD with Remote Execution

```yaml
# .github/workflows/deploy.yml
# Remote execution means the actual plan/apply runs in Terraform Cloud
# GitHub Actions just triggers and monitors the run

name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: '1.7.0'
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: OpenTofu Init
        run: tofu init

      - name: OpenTofu Plan (runs in Terraform Cloud)
        run: tofu plan -no-color
        env:
          TF_INPUT: false

      - name: OpenTofu Apply (runs in Terraform Cloud)
        run: tofu apply -auto-approve -no-color
        env:
          TF_INPUT: false
```

## Run Output and Exit Codes

```bash
# Remote execution exit codes match local execution:
# 0 = success (plan: no changes; apply: success)
# 1 = error
# 2 = plan: changes present (with -detailed-exitcode)

# Capture exit code in scripts
tofu plan -detailed-exitcode
EXIT_CODE=$?

case $EXIT_CODE in
  0) echo "No changes" ;;
  1) echo "Error" ; exit 1 ;;
  2) echo "Changes pending - running apply" ; tofu apply -auto-approve ;;
esac
```

## Conclusion

Remote execution with the cloud backend uploads your configuration to Terraform Cloud, runs the plan or apply on managed infrastructure with workspace-injected credentials, and streams output back to your terminal. The key benefit is credential isolation - AWS/Azure/GCP credentials live only in Terraform Cloud workspace variables, never on developer machines or in CI/CD systems. All runs are logged, with output available in the Terraform Cloud UI for auditing.
