# How to Migrate from Local Backend to Cloud Backend in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Migration, Cloud Backend, State Migration, Terraform Cloud

Description: Learn how to safely migrate OpenTofu state from a local backend to the Terraform Cloud backend, including preparation steps, migration execution, and verification.

## Introduction

Migrating from a local state file to the Terraform Cloud backend enables remote state storage, state locking, team collaboration, and run history. The migration copies your existing state to Terraform Cloud and reconfigures OpenTofu to use the cloud backend. The process is reversible if needed.

## Pre-Migration Checklist

```bash
# 1. Verify current state is clean
tofu state list
tofu plan  # Should show no changes

# 2. Backup current state
cp terraform.tfstate terraform.tfstate.backup-$(date +%Y%m%d)
cp terraform.tfstate.backup terraform.tfstate.backup.backup-$(date +%Y%m%d) 2>/dev/null || true

# 3. Ensure no runs are in progress
# If using remote state (S3, GCS, etc.), check for locks

# 4. Create Terraform Cloud workspace before migration
# Log into app.terraform.io and create the workspace
# (or use the API)

# 5. Authenticate with Terraform Cloud
tofu login
```

## Creating the Destination Workspace

```bash
# Create workspace via Terraform Cloud API
curl -X POST \
  -H "Authorization: Bearer $TF_TOKEN" \
  -H "Content-Type: application/vnd.api+json" \
  "https://app.terraform.io/api/v2/organizations/my-company/workspaces" \
  -d '{
    "data": {
      "type": "workspaces",
      "attributes": {
        "name": "production-infrastructure",
        "description": "Migrated from local state",
        "terraform-version": "1.7.0",
        "execution-mode": "local",
        "auto-apply": false
      }
    }
  }'
```

## Step 1: Update Backend Configuration

```hcl
# Before: local backend (or no backend = local by default)
# terraform {
#   backend "local" {
#     path = "terraform.tfstate"
#   }
# }

# After: cloud backend
terraform {
  cloud {
    organization = "my-company"

    workspaces {
      name = "production-infrastructure"
    }
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Step 2: Run Migration

```bash
# Run init - OpenTofu detects backend change and prompts for migration
tofu init

# Expected output:
# Initializing Terraform Cloud...
#
# Terraform Cloud has been successfully initialized!
#
# The new backend configuration doesn't match the old state file
# in the current directory.
#
# To use the new backend, you must first migrate the state.
# Do you want to migrate your state? (yes/no): yes
#
# Acquiring state lock. This may take a few moments...
# Migrating the latest state snapshot to the new backend...
# Successfully migrated the state to Terraform Cloud!

# Verify migration
tofu state list  # Should show same resources
```

## Step 3: Verify Migration

```bash
# Verify state in Terraform Cloud
tofu plan  # Should show no changes (state matches infrastructure)

# Check state in Terraform Cloud UI:
# Workspace → States → (should show your resources)

# Verify state count matches
LOCAL_COUNT=$(tofu state list | wc -l)
echo "Resources in state: $LOCAL_COUNT"

# View current state
tofu show
```

## Migrating from S3 Backend to Cloud Backend

```hcl
# Before: S3 backend
# terraform {
#   backend "s3" {
#     bucket = "my-tfstate-bucket"
#     key    = "production/terraform.tfstate"
#     region = "us-east-1"
#   }
# }

# After: cloud backend
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
# For remote backends (S3, GCS, Azure Blob):
# OpenTofu pulls state from the remote backend and pushes to Terraform Cloud

tofu init
# "Do you want to migrate your state? (yes/no): yes"
# Reads from S3, writes to Terraform Cloud

# Verify no changes
tofu plan  # Should show: No changes
```

## Handling Multiple State Files

```bash
#!/bin/bash
# migrate-workspaces.sh - Migrate multiple workspaces

WORKSPACES=("development" "staging" "production")
TF_DIR="./infrastructure"

for WS in "${WORKSPACES[@]}"; do
  echo "Migrating workspace: $WS"

  # Update workspace name in config
  sed -i "s/name = \".*\"/name = \"infrastructure-${WS}\"/" \
    "$TF_DIR/backend.tf"

  cd "$TF_DIR"

  # Switch to correct local workspace
  tofu workspace select "$WS" 2>/dev/null || \
    tofu workspace new "$WS"

  # Migrate this workspace's state
  tofu init -migrate-state -force-copy

  echo "Migrated $WS successfully"
  cd -
done
```

## Post-Migration Cleanup

```bash
# After successful migration and verification:

# 1. Remove local state files
# (Keep backups for a few weeks before deleting)
mv terraform.tfstate /tmp/terraform.tfstate.pre-migration.$(date +%Y%m%d)

# 2. Remove .terraform directory to clean up old backend config
rm -rf .terraform/

# 3. Re-initialize with cloud backend
tofu init

# 4. Set workspace execution mode (local or remote)
# (Set to local if you need local execution environment)

# 5. Add workspace variables if switching to remote execution
# AWS credentials, etc.

# 6. Verify final state
tofu plan  # Must show no changes
```

## Rolling Back if Migration Fails

```bash
# If migration fails or causes issues:

# 1. Remove the cloud backend configuration
# Edit main.tf to remove the cloud {} block

# 2. Restore local state from backup
cp terraform.tfstate.backup-20240101 terraform.tfstate

# 3. Re-initialize with local backend
tofu init  # Detects revert, prompts to migrate back

# 4. Verify rollback
tofu state list
tofu plan  # Should show no changes
```

## Conclusion

Migrating from local to cloud backend is a one-command operation: update the `terraform {}` block to include `cloud {}` configuration, then run `tofu init`. OpenTofu detects the backend change, prompts for confirmation, reads the existing state (local file or remote backend), and writes it to Terraform Cloud. Always back up your state file before migration and verify with `tofu plan` — it should show no changes — before committing to the new backend. The rollback process is symmetric: revert the configuration and run `tofu init` again.
