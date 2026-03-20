# How to Use Taint and Untaint in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Resources, Taint, Untaint, Infrastructure as Code, DevOps

Description: A guide to using taint and untaint commands in OpenTofu to mark resources for forced recreation or remove that marking.

## Introduction

The `tofu taint` command marks a resource in the state file for forced recreation on the next apply. The `tofu untaint` command removes that marking. While `tofu apply -replace` is now preferred for most use cases, understanding taint is useful for existing workflows and situations where you need to mark resources without immediately applying.

## Basic taint Usage

```bash
# Mark a resource for replacement
tofu taint aws_instance.web

# Output:
# Resource instance aws_instance.web has been marked as tainted.
```

## Untaint to Remove the Marking

```bash
# Remove the taint marking (cancel planned replacement)
tofu untaint aws_instance.web

# Output:
# Resource instance aws_instance.web has been successfully untainted.
```

## Taint vs -replace Flag

```bash
# Taint: modifies state file, replacement happens on next apply
tofu taint aws_instance.web
tofu apply  # Will recreate web instance

# -replace flag: atomic plan + apply in one command (preferred)
tofu apply -replace="aws_instance.web"

# When to use taint over -replace:
# 1. When you want to schedule replacement for a later apply
# 2. When working with CI/CD pipelines that run separate plan/apply
# 3. Legacy workflows that expect taint behavior
```

## Taint with Resource Indices

```bash
# Taint a count-indexed resource
tofu taint 'aws_instance.web[0]'
tofu taint 'aws_instance.web[2]'

# Taint a for_each resource
tofu taint 'aws_instance.web["us-east-1a"]'

# Taint a module resource
tofu taint 'module.app.aws_instance.server'
tofu taint 'module.workers.aws_instance.worker[1]'
```

## Listing Tainted Resources

```bash
# Check if any resources are tainted
tofu show

# Or use state list to see all resources, then check each
tofu state list

# Look for "tainted" in state show output
tofu state show aws_instance.web
# The state show output indicates tainted status
```

## Practical Use Cases

```bash
# Use case 1: Degraded EC2 instance
tofu taint aws_instance.app_server
# Now the instance will be replaced on next tofu apply
# Can be done during business hours prep, apply during maintenance window

# Use case 2: Failed deployment left instance in bad state
tofu taint aws_instance.deployment_target
tofu apply

# Use case 3: Need to refresh Kubernetes node
tofu taint 'module.eks_nodes.aws_autoscaling_group.workers'
tofu apply

# Use case 4: Certificate needs rotation
tofu taint aws_acm_certificate.main
tofu apply
```

## Untaint Use Cases

```bash
# You tainted the wrong resource
tofu taint aws_instance.web_1  # Oops, meant web_2
tofu untaint aws_instance.web_1
tofu taint aws_instance.web_2

# The planned maintenance was cancelled
tofu taint aws_db_instance.main  # Marked for replacement
# Maintenance cancelled
tofu untaint aws_db_instance.main

# Verify untaint worked
tofu plan  # Should show no replacements planned
```

## Taint in Automated Workflows

```bash
#!/bin/bash
# Legacy CI/CD script using taint

# Mark resource for replacement
tofu taint "${RESOURCE_TO_REPLACE}"

# Plan and save
tofu plan -out=tainted-plan.tfplan

# Review plan (in CI, this might be a manual approval gate)
tofu show tainted-plan.tfplan

# Apply
tofu apply tainted-plan.tfplan
```

## State File Implications

```bash
# Taint modifies the state file directly
# In team environments, state locking prevents conflicts

# Check state after taint
tofu state pull | python3 -c "
import json, sys
state = json.load(sys.stdin)
for resource in state.get('resources', []):
    for instance in resource.get('instances', []):
        if instance.get('status') == 'tainted':
            print(f\"Tainted: {resource['type']}.{resource['name']}\")
"
```

## Modern Alternative: -replace Flag

```bash
# For new workflows, prefer -replace over taint
# It's atomic and doesn't modify state before the operation

# Old workflow:
tofu taint aws_instance.web
tofu apply

# New workflow (preferred):
tofu apply -replace="aws_instance.web"

# For plan/apply separation:
tofu plan -replace="aws_instance.web" -out=replace.tfplan
tofu apply replace.tfplan
```

## Conclusion

The `tofu taint` and `tofu untaint` commands provide a way to schedule forced resource recreation for a later apply operation. While `tofu apply -replace` is generally preferred for its atomicity, taint remains useful in workflows where planning and applying are separate steps. Understanding taint also helps when working with older OpenTofu/Terraform configurations that use this pattern. Use `tofu untaint` to cancel a planned replacement if circumstances change before the next apply.
