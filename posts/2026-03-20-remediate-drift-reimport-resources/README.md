# How to Remediate Drift by Re-Importing Resources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Drift Remediation, Import, State Management, Infrastructure as Code, Best Practices

Description: Learn how to use tofu import to reconcile drifted resources when state has become desynchronized from the actual cloud configuration.

## Introduction

Re-importing is the appropriate drift remediation when a resource's state entry is completely missing or severely corrupted. Unlike `apply -refresh-only` (which updates attribute values), import brings a resource back under OpenTofu management when it has fallen out of state entirely.

## When to Use Re-Import

- Resource was manually deleted from state (`tofu state rm`) but still exists in the cloud
- State file was corrupted or lost and needs to be rebuilt
- Resource was created outside OpenTofu and you now want to manage it
- A migration left state entries orphaned or pointing to wrong resource IDs

## Import Workflow

```bash
# Step 1: Verify the resource exists in the cloud

aws ec2 describe-instances --instance-ids i-0abc123def456789

# Step 2: Check current state for the resource
tofu state list | grep "aws_instance.web"
# If empty → resource needs to be imported

# Step 3: Import the resource
tofu import aws_instance.web i-0abc123def456789

# Step 4: Verify the import
tofu state show aws_instance.web

# Step 5: Run plan to confirm no spurious changes
tofu plan
# Expected: No changes (or only differences to reconcile)
```

## Declarative Import with import Block (OpenTofu 1.5+)

```hcl
# import.tf - declare imports in configuration
import {
  to = aws_instance.web
  id = "i-0abc123def456789"
}

import {
  to = aws_db_instance.main
  id = "prod-postgres"
}

import {
  to = aws_vpc.main
  id = "vpc-0abc123def456789"
}
```

```bash
# Apply the imports
tofu plan    # Shows what will be imported
tofu apply   # Performs the imports

# Remove import blocks after apply - they are one-time operations
```

## Generating Configuration from an Existing Resource

OpenTofu 1.6+ can generate configuration for imported resources:

```bash
# Generate HCL configuration for an existing resource
tofu plan -generate-config-out=generated.tf

# Review the generated configuration
cat generated.tf

# Edit as needed, then run plan again
tofu plan
```

## Bulk Re-Import After State Loss

If you lost an entire state file, script the imports:

```bash
#!/bin/bash
# rebuild-state.sh - re-import after state loss

# Import all EC2 instances with specific tag
aws ec2 describe-instances \
  --filters "Name=tag:ManagedBy,Values=opentofu" \
  --query "Reservations[*].Instances[*].[InstanceId,Tags[?Key=='Name'].Value|[0]]" \
  --output text | while read instance_id name; do
    resource_name=$(echo "$name" | tr '-' '_' | tr '[:upper:]' '[:lower:]')
    echo "Importing aws_instance.${resource_name} ${instance_id}"
    tofu import "aws_instance.${resource_name}" "$instance_id"
done
```

## Reconciling Configuration After Import

After import, run plan to find discrepancies:

```bash
tofu plan 2>&1 | grep -E "~|forces replacement"
```

If the plan shows changes to reconcile:

```hcl
# Update configuration to match the imported resource's actual state
resource "aws_instance" "web" {
  # Update these attributes to match what was imported
  instance_type = "m5.large"  # Actual value from cloud, may differ from config
  ami           = "ami-0abc123"
}
```

## Conclusion

Re-importing fixes drift when a resource has fallen out of state entirely. Use the declarative `import` block for documented, reviewable imports, and `tofu plan -generate-config-out` to generate a starting configuration for complex resources. After any import, run `tofu plan` to identify configuration differences that need to be reconciled to achieve a clean no-changes state.
