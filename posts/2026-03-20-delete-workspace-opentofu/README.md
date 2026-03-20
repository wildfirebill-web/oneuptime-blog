# How to Delete a Workspace in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Workspaces

Description: Learn how to safely delete an OpenTofu workspace after destroying its infrastructure, including the proper sequence to avoid orphaned resources.

## Introduction

When a workspace is no longer needed - for example, when a feature branch environment is decommissioned - you should delete it to keep your workspace list clean and avoid confusion. Deleting a workspace requires first destroying any managed infrastructure, then removing the workspace.

## Prerequisites

Before deleting a workspace:
1. The workspace must not be the currently active workspace
2. The workspace's state must be empty (all resources destroyed)
3. You cannot delete the `default` workspace

## Step 1: Destroy Infrastructure in the Workspace

```bash
# Switch to the workspace you want to delete

tofu workspace select dev-feature-x

# Verify what's managed in this workspace
tofu state list

# Destroy all infrastructure
tofu destroy -var-file=dev.tfvars

# Confirm the state is now empty
tofu state list
# (should return nothing)
```

## Step 2: Switch to a Different Workspace

You cannot delete the currently active workspace:

```bash
# Switch away from the workspace you want to delete
tofu workspace select default

# Or any other workspace
tofu workspace select staging
```

## Step 3: Delete the Workspace

```bash
# Delete the workspace
tofu workspace delete dev-feature-x

# Output:
# Deleted workspace "dev-feature-x"!
```

## Deleting a Workspace with Remaining State

If you try to delete a workspace that still has resources:

```bash
tofu workspace delete dev-feature-x
# Error: Workspace "dev-feature-x" is not empty.
#
# Deleting "dev-feature-x" can result in dangling resources: resources that
# exist in your cloud infrastructure but are no longer managed by OpenTofu.
# Please destroy these resources first. If you want to delete this workspace
# anyway, use the -force flag.
```

### Force Delete (Use with Caution)

```bash
# Force delete - leaves infrastructure in place (orphaned resources)
tofu workspace delete -force dev-feature-x

# CAUTION: Resources managed by this workspace are now unmanaged
# You'll need to manually track or delete them
```

## Cleanup Automation

Script to clean up old development workspaces:

```bash
#!/bin/bash
# cleanup-workspaces.sh

WORKSPACE_TO_DELETE="$1"

if [ -z "$WORKSPACE_TO_DELETE" ]; then
  echo "Usage: $0 <workspace-name>"
  exit 1
fi

if [ "$WORKSPACE_TO_DELETE" = "default" ]; then
  echo "Cannot delete the default workspace"
  exit 1
fi

echo "Preparing to delete workspace: $WORKSPACE_TO_DELETE"

# Switch to the target workspace
tofu workspace select "$WORKSPACE_TO_DELETE"

# Check if there are resources
RESOURCE_COUNT=$(tofu state list 2>/dev/null | wc -l)

if [ "$RESOURCE_COUNT" -gt 0 ]; then
  echo "Found $RESOURCE_COUNT resources. Destroying..."
  tofu destroy -auto-approve

  # Verify destruction
  NEW_COUNT=$(tofu state list 2>/dev/null | wc -l)
  if [ "$NEW_COUNT" -gt 0 ]; then
    echo "ERROR: $NEW_COUNT resources still exist after destroy"
    exit 1
  fi
fi

# Switch to default and delete
tofu workspace select default
tofu workspace delete "$WORKSPACE_TO_DELETE"

echo "Workspace $WORKSPACE_TO_DELETE successfully deleted"
tofu workspace list
```

## Deleting Backend State Files Directly

If force deletion is used, clean up the orphaned state file manually:

```bash
# For S3 backend
aws s3 rm s3://my-state-bucket/env:/dev-feature-x/terraform.tfstate

# For local backend
rm -rf terraform.tfstate.d/dev-feature-x/

# For GCS backend
gsutil rm gs://my-state-bucket/prod/dev-feature-x.tfstate
```

## Preventing Accidental Deletion

Add a check in your CI/CD pipeline:

```bash
# Check that a workspace doesn't have active resources before deletion
check_workspace_empty() {
  local ws="$1"
  tofu workspace select "$ws"
  count=$(tofu state list | wc -l)
  tofu workspace select default
  echo $count
}

RESOURCE_COUNT=$(check_workspace_empty "dev-feature-x")
if [ "$RESOURCE_COUNT" -gt 0 ]; then
  echo "ERROR: Workspace still has $RESOURCE_COUNT resources"
  exit 1
fi
```

## Conclusion

Deleting an OpenTofu workspace is a two-step process: first destroy the infrastructure, then delete the workspace. The `default` workspace cannot be deleted. Use force deletion only when you consciously want to abandon resources - always prefer proper `tofu destroy` followed by clean workspace deletion. Regular workspace cleanup keeps your environment list manageable and prevents confusion.
