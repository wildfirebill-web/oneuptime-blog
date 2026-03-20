# How to Use Local Execution with Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Cloud Backend, Local Execution, Terraform Cloud, State Management

Description: Learn how to configure local execution mode with the OpenTofu cloud backend to run plans and applies locally while using Terraform Cloud for state storage and locking.

## Introduction

Local execution mode with the cloud backend runs `tofu plan` and `tofu apply` on your local machine (or CI/CD runner) but stores state in Terraform Cloud and uses Terraform Cloud's state locking. This combines the flexibility of local execution (access to local files, network, custom tools) with Terraform Cloud's collaborative state management.

## Configuring Local Execution Mode

```hcl
# main.tf
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
# Set workspace execution mode to Local in Terraform Cloud UI:
# Workspace → Settings → General → Execution Mode → Local

# Or via API:
curl -X PATCH \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/workspaces/$WORKSPACE_ID" \
  -d '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "execution-mode": "local"
      }
    }
  }'
```

## Running Locally with Cloud State

```bash
# Set credentials for the cloud backend
export TF_TOKEN_app_terraform_io="your-terraform-cloud-token"

# Set cloud credentials locally (used during local execution)
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_DEFAULT_REGION="us-east-1"

# Init connects to cloud backend for state
tofu init

# Plan runs locally, state stored in Terraform Cloud
tofu plan

# Apply runs locally, writes state to Terraform Cloud
tofu apply
```

## Use Cases for Local Execution

```bash
# 1. Access to local files not uploadable to Terraform Cloud
#    (large binary files, local certificates, etc.)
tofu apply -var="cert_file=$(cat /etc/ssl/certs/custom.pem)"

# 2. Access to internal networks from CI/CD runners
#    (database migrations, internal API calls)
resource "null_resource" "db_migration" {
  provisioner "local-exec" {
    command = "psql ${var.db_url} -f migrations/001_init.sql"
  }
}

# 3. Custom provider binaries not in the public registry
# Place the binary in .terraform/providers/ directory
# Local execution uses the local binary

# 4. Development and debugging
# Faster iteration without upload overhead
```

## State Locking with Local Execution

```bash
# Terraform Cloud provides state locking even in local mode
# If two engineers run tofu apply simultaneously:

# Engineer A:
tofu apply
# Acquiring state lock... (succeeds)
# Applying...

# Engineer B (at the same time):
tofu apply
# Acquiring state lock... (waiting)
# Error: Error acquiring the state lock
# State is locked by another process (timeout after 30s)
```

## Comparing Local vs Remote Execution

```bash
# Local execution advantages:
# + Access to local network resources
# + No file upload size limits
# + Use local provider binaries
# + Faster for small changes (no upload time)
# + Local environment variables available

# Remote execution advantages:
# + Credentials never leave Terraform Cloud
# + Consistent execution environment
# + Plans visible in Terraform Cloud UI
# + Policy enforcement (Sentinel/OPA)
# + Run queue and concurrency controls
```

## Hybrid Approach: Local for Dev, Remote for Prod

```hcl
# Use workspace naming convention for execution mode
# development workspace: local execution
# production workspace: remote execution

# dev/main.tf
terraform {
  cloud {
    organization = "my-company"
    workspaces {
      name = "development-infrastructure"
    }
  }
}

# prod/main.tf
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
# Set workspace execution modes via script
for WORKSPACE in development-infrastructure staging-infrastructure; do
  WORKSPACE_ID=$(get_workspace_id "$WORKSPACE")
  set_execution_mode "$WORKSPACE_ID" "local"
done

for WORKSPACE in production-infrastructure; do
  WORKSPACE_ID=$(get_workspace_id "$WORKSPACE")
  set_execution_mode "$WORKSPACE_ID" "remote"
done
```

## CI/CD with Local Execution Mode

```yaml
# .github/workflows/deploy.yml
# Local execution: plan and apply run on GitHub Actions runner
# State stored in Terraform Cloud

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          cli_config_credentials_token: ${{ secrets.TF_API_TOKEN }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1

      - name: OpenTofu Init
        run: tofu init
        # Connects to Terraform Cloud for state, but runs locally

      - name: OpenTofu Apply
        run: tofu apply -auto-approve
        # Runs locally using GitHub Actions AWS credentials
        # Writes state to Terraform Cloud
```

## Viewing State in Terraform Cloud UI

```bash
# Even with local execution, runs appear in Terraform Cloud
# but as "local plan" and "local apply" entries

# State versions are visible in:
# Workspace → States → (list of state versions with timestamps)

# Download state from Terraform Cloud
tofu state pull > current.tfstate

# Force-push corrected state (use carefully)
tofu state push corrected.tfstate
```

## Conclusion

Local execution mode combines the best of both worlds: Terraform Cloud provides state storage, versioning, and locking, while plans and applies run on your machine or CI/CD runner with access to local resources and network. This mode is ideal when you need local network access (for provisioners or internal API calls), have large files that can't be uploaded, or are debugging with custom provider binaries. Set the execution mode per-workspace based on the environment's security and access requirements.
