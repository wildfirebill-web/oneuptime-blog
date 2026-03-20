---
title: "Using tofu state push in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, state
description: "Learn how to use tofu state push to upload a local state file to your configured remote backend, and when this advanced operation is appropriate."
---

# Using tofu state push in OpenTofu

The `tofu state push` command uploads a local state file to the configured remote backend. It's an advanced operation used in specific recovery and migration scenarios. Use it with extreme caution — uploading incorrect state can cause serious problems.

## When to Use state push

`state push` is appropriate for:
- **Recovering from corruption**: Restore a known-good backup
- **Backend migration**: Move state from one backend to another
- **Initial setup**: Seed remote backend with a local state file
- **Emergency fixes**: Correct state after a failed operation

**Avoid using state push for**:
- Routine operations (use normal apply workflows)
- Bypassing locking without understanding the risk
- Making manual edits to state (use dedicated state commands instead)

## Basic Usage

```bash
# Upload a local state file to the configured backend
tofu state push terraform.tfstate

# Output:
# Pushing state to remote backend...
# State successfully pushed.
```

## State Serial Validation

OpenTofu validates that the state you're pushing has a higher serial number than the current state (to prevent accidental rollbacks):

```bash
tofu state push backup.tfstate
# Error: Remote state version is newer than the local state
# ...

# Check serial numbers
tofu state pull | jq .serial
cat backup.tfstate | jq .serial

# If backup has lower serial, use -force to override (DANGEROUS)
tofu state push -force backup.tfstate
```

## Backend Migration Workflow

```bash
# Scenario: migrate from local state to S3

# Step 1: You have local terraform.tfstate
ls terraform.tfstate

# Step 2: Configure the new backend
cat >> main.tf << 'EOF'

terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "production/terraform.tfstate"
    region = "us-east-1"
  }
}
EOF

# Step 3: Initialize (OpenTofu will ask to migrate automatically)
tofu init
# Do you want to copy existing state to the new backend? yes

# OpenTofu handles this automatically — you don't need state push
# state push is for when you need manual control
```

## Manual Backend Migration

For cases where automatic migration doesn't work:

```bash
# Step 1: Pull current state
tofu state pull > local-backup.tfstate

# Step 2: Reconfigure backend
# (edit main.tf to point to new backend)

# Step 3: Initialize new backend
tofu init

# Step 4: Push state to new backend
tofu state push local-backup.tfstate

# Step 5: Verify
tofu state pull | jq .serial
tofu plan  # Should show no changes
```

## State Recovery from Backup

```bash
# Scenario: state was corrupted, need to restore backup

# Step 1: Check available backups
ls -la backups/
# backups/terraform.tfstate.20260301-120000
# backups/terraform.tfstate.20260301-090000

# Step 2: Inspect the backup
cat backups/terraform.tfstate.20260301-120000 | jq .serial

# Step 3: Compare with current state
tofu state pull | jq .serial

# Step 4: If backup is valid and newer or appropriate
# Check if it's safe to restore
cat backups/terraform.tfstate.20260301-120000 | jq '.resources | length'

# Step 5: Bump the serial to be newer than current (if needed)
cat backups/terraform.tfstate.20260301-120000 | jq '.serial += 1' > /tmp/fixed.tfstate

# Step 6: Push the backup
tofu state push /tmp/fixed.tfstate

# Step 7: Verify
tofu plan
```

## The -force Flag

```bash
# WARNING: -force bypasses serial number validation
# Only use when you're absolutely sure about what you're pushing

tofu state push -force older-state.tfstate

# This can cause:
# - Loss of recent state changes
# - Resources being created or destroyed unexpectedly
# - Configuration drift

# ALWAYS run plan after force push
tofu plan
```

## Seeding a New Remote Backend

```bash
# Scenario: starting fresh with remote state for an existing infra

# Step 1: Import existing resources to build state
tofu import aws_instance.web i-0123456789abcdef0
tofu import aws_vpc.main vpc-0abc12345

# Step 2: After building state locally
# State is in terraform.tfstate

# Step 3: Configure remote backend
# (add backend block to main.tf)

# Step 4: Init and migrate
tofu init
# OpenTofu will offer to copy state — say yes

# If it doesn't offer:
tofu state push terraform.tfstate
```

## Comparing Before Pushing

```bash
# Before pushing, compare what you're about to push
current_resources=$(tofu state pull | jq '.resources | length')
push_resources=$(cat local.tfstate | jq '.resources | length')

echo "Current resources in backend: $current_resources"
echo "Resources in file to push:    $push_resources"

if [[ "$push_resources" -lt "$current_resources" ]]; then
  echo "WARNING: You're pushing a state with FEWER resources!"
  echo "This may cause OpenTofu to destroy resources on next apply"
  read -p "Continue? (yes/no): " confirm
  [[ "$confirm" == "yes" ]] || exit 1
fi

tofu state push local.tfstate
```

## Conclusion

`tofu state push` is a low-level tool reserved for recovery and migration scenarios. In normal operations, let OpenTofu manage state automatically through `apply` and `init -migrate-state`. When you do need to push state, always verify the serial numbers, back up before making changes, use dry-runs where possible, and always run `tofu plan` after pushing to verify the state matches your infrastructure.
