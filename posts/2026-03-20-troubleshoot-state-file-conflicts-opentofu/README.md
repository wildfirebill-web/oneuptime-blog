# How to Troubleshoot State File Conflicts in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Troubleshooting, State, Conflicts, Infrastructure as Code

Description: Learn how to diagnose and resolve state file conflicts, including stale locks, concurrent apply errors, and state drift caused by out-of-band changes.

## Introduction

State file conflicts arise when multiple processes attempt to modify the same state simultaneously, when a previous run crashed leaving a stale lock, or when manual changes outside OpenTofu cause state drift. Understanding how to safely resolve each scenario prevents data loss and infrastructure inconsistency.

## Error: State Locked by Another Process

```text
Error: Error acquiring the state lock
Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        3f5c9a2e-1b4d-4e8f-a7c1-9d2e3f4a5b6c
  Path:      my-tofu-state/prod/terraform.tfstate
  Operation: OperationTypeApply
  Who:       user@machine
  Created:   2026-03-20 09:15:22.123456789 +0000 UTC
```

First, verify the lock is actually stale (the process is no longer running).

```bash
# Check if the locking process is still alive

# The "Who" field shows user@machine - check that machine
ps aux | grep tofu

# For DynamoDB locks (legacy), check the lock table
aws dynamodb scan \
  --table-name tofu-state-locks \
  --query 'Items[*].[LockID.S,Info.S]' \
  --output table

# For native S3 locking (OpenTofu 1.10+), check for .tflock file
aws s3 ls s3://my-tofu-state/prod/ | grep tflock
```

Only force-unlock after confirming no other process is running.

```bash
# Force unlock using the lock ID from the error message
tofu force-unlock 3f5c9a2e-1b4d-4e8f-a7c1-9d2e3f4a5b6c

# For native S3 locking, delete the lock file directly
aws s3 rm s3://my-tofu-state/prod/terraform.tfstate.tflock
```

## Error: State File Corrupted or Empty

```text
Error: Failed to load state: state file invalid
```

```bash
# Pull the current state and inspect it
tofu state pull > current-state.json
cat current-state.json | python3 -m json.tool | head -50

# If the state is empty or malformed, restore from a backup
# S3 with versioning - list previous versions
aws s3api list-object-versions \
  --bucket my-tofu-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified,IsLatest]' \
  --output table

# Restore a previous version
aws s3api get-object \
  --bucket my-tofu-state \
  --key prod/terraform.tfstate \
  --version-id <version-id> \
  restored-state.json

# Push the restored state
tofu state push restored-state.json
```

Always enable S3 versioning on your state bucket - it's your safety net.

```hcl
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## State Drift: Resources Changed Outside OpenTofu

When someone manually modifies a resource in the console or CLI, the state no longer matches reality.

```bash
# Refresh state to detect drift
tofu refresh

# Then run plan to see what OpenTofu wants to change
tofu plan

# If you want to accept the manual change (update state to match reality):
tofu import aws_security_group.web sg-0123456789abcdef0

# If you want to revert the manual change (restore to state):
tofu apply  # will revert to the declared configuration
```

## Conflict: Two Teams Applied Simultaneously

When two teams both run `tofu apply` against the same state at the same time (race condition during lock acquisition), one succeeds and one fails.

```bash
# After the failed apply, check what actually changed
tofu plan

# The plan will show what the successful apply did vs. your configuration
# Review carefully before applying again
```

Prevent this with a CI/CD gate that serializes applies.

```yaml
# GitHub Actions: serialize applies with concurrency
jobs:
  apply:
    concurrency:
      group: tofu-prod-apply
      cancel-in-progress: false  # queue, don't cancel
```

## State Resource Address Conflicts

When modules are refactored, resource addresses change and OpenTofu tries to destroy and recreate resources.

```bash
# Move state to the new address without destroying
tofu state mv \
  module.old_name.aws_instance.web \
  module.new_name.aws_instance.web

# Move multiple resources with a script
tofu state list | grep "module.old_name" | while read addr; do
  new_addr=$(echo "$addr" | sed 's/module.old_name/module.new_name/')
  tofu state mv "$addr" "$new_addr"
done

# Verify the move
tofu plan  # should show no changes
```

## Summary

State conflicts fall into four categories: stale locks (verify the process is dead, then `tofu force-unlock`), corrupted state (restore from S3 versioned backup with `tofu state push`), state drift (use `tofu refresh` and then `tofu plan` to understand the delta), and address conflicts from refactoring (use `tofu state mv` to remap without destroying). Always enable S3 versioning and use a CI/CD concurrency group to serialize applies - prevention is far cheaper than recovery.
