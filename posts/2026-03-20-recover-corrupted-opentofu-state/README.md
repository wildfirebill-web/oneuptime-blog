# How to Recover a Corrupted OpenTofu State File

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, State Management

Description: Learn step-by-step how to recover a corrupted OpenTofu state file by restoring from backup, manually editing, or rebuilding state from scratch.

## Introduction

A corrupted OpenTofu state file can bring your infrastructure management to a halt. Corruption can occur due to interrupted writes, manual edits gone wrong, or storage issues. This guide covers multiple recovery strategies from simplest to most involved.

## Signs of a Corrupted State File

Common symptoms include:

```
Error: Failed to load state: state snapshot was created by an incompatible version
Error: The state file could not be loaded: unexpected token
panic: runtime error: index out of range
```

## Step 1: Never Modify Real Infrastructure Until State is Restored

Before taking any action, ensure that no one runs `tofu apply` or `tofu destroy`. A corrupted state with running infrastructure could lead to resource duplication or deletion.

```bash
# Immediately lock the state if possible, or communicate to your team
# Do NOT run tofu apply until the state is healthy
```

## Step 2: Restore from Backup

The safest recovery method is restoring a recent backup.

### Restoring from S3 Versioning

```bash
# List versions of the state file in S3
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix prod/terraform.tfstate \
  --query 'Versions[*].[VersionId,LastModified]' \
  --output table

# Download a specific version
aws s3api get-object \
  --bucket my-terraform-state \
  --key prod/terraform.tfstate \
  --version-id "abc123EXAMPLE" \
  terraform.tfstate.backup

# Copy it back as the current state
aws s3 cp terraform.tfstate.backup \
  s3://my-terraform-state/prod/terraform.tfstate
```

### Restoring a Local Backup

```bash
# OpenTofu creates backup files before each apply
ls -la *.tfstate*

# Restore the backup
cp terraform.tfstate.backup terraform.tfstate
```

## Step 3: Validate the Restored State

After restoring, verify the state is valid:

```bash
# Check state validity by listing resources
tofu state list

# Run a plan to see if state matches actual infrastructure
tofu plan
```

## Step 4: Manual State File Repair

If no backup is available, you can manually edit the state file. The state file is valid JSON:

```bash
# First, make a copy of the corrupted file
cp terraform.tfstate terraform.tfstate.corrupted

# Validate the JSON structure
python3 -m json.tool terraform.tfstate > /dev/null

# View the structure
cat terraform.tfstate | python3 -m json.tool | head -50
```

A valid state file follows this structure:

```json
{
  "version": 4,
  "terraform_version": "1.8.0",
  "serial": 42,
  "lineage": "abc12345-...",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.opentofu.org/hashicorp/aws\"]",
      "instances": [...]
    }
  ]
}
```

## Step 5: Rebuild State with tofu import

If the state cannot be recovered, rebuild it by importing existing resources:

```hcl
# Use import blocks (OpenTofu 1.7+) to re-import all resources
import {
  to = aws_vpc.main
  id = "vpc-0a1b2c3d4e5f"
}

import {
  to = aws_instance.web
  id = "i-0123456789abcdef0"
}
```

```bash
# Apply the imports to rebuild the state
tofu plan    # Review what will be imported
tofu apply   # Import the resources
```

## Step 6: Verify Infrastructure Consistency

After recovery, run a full plan and confirm there are no unexpected changes:

```bash
tofu plan -out=recovery.tfplan

# Review the plan carefully before applying
tofu show recovery.tfplan
```

Only apply if the plan shows no unintended changes (infrastructure should match your configuration).

## Prevention: Enable State Versioning

```hcl
# S3 backend with versioning enabled
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

# Enable point-in-time recovery
resource "aws_s3_bucket_lifecycle_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    id     = "retain-state-versions"
    status = "Enabled"

    noncurrent_version_expiration {
      noncurrent_days = 90  # Keep 90 days of history
    }
  }
}
```

## Conclusion

Recovering a corrupted OpenTofu state file requires quick action and a methodical approach. Restoring from a versioned backup is always the fastest path. When that's not available, manual JSON repair or re-importing resources from scratch are viable options. The best defense is prevention: always enable S3 versioning or equivalent backup mechanisms for your state backend.
