---
title: "Using tofu state rm in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, state
description: "Learn how to use tofu state rm to remove resources from OpenTofu state without destroying the actual infrastructure."
---

# Using tofu state rm in OpenTofu

The `tofu state rm` command removes one or more resources from the state file without destroying the actual infrastructure. The real resource continues to exist in your cloud provider — OpenTofu simply stops tracking it.

## When to Use state rm

- **Abandon management**: You want OpenTofu to stop managing a resource
- **Migrate to another workspace**: Move resource management elsewhere
- **Import elsewhere**: Remove before importing into a different configuration
- **Emergency cleanup**: Fix a broken state without destroying infrastructure
- **Remove dependent resources**: Before removing a resource from config

## Basic Usage

```bash
# Remove a single resource from state
tofu state rm aws_instance.old_web

# Output:
# Removed aws_instance.old_web
# Successfully removed 1 resource instance(s).
```

After this, the EC2 instance still exists in AWS, but OpenTofu won't manage it.

## Removing Module Resources

```bash
# Remove a resource inside a module
tofu state rm 'module.networking.aws_subnet.legacy'

# Remove all resources in a module
tofu state rm 'module.deprecated_module'
# This removes all resources within the module from state
```

## Removing count Resources

```bash
# Remove a specific count index
tofu state rm 'aws_instance.app[2]'

# Remove all instances of a count resource
tofu state rm 'aws_instance.app[0]'
tofu state rm 'aws_instance.app[1]'
tofu state rm 'aws_instance.app[2]'
```

## Removing for_each Resources

```bash
# Remove a specific for_each instance
tofu state rm 'aws_s3_bucket.regional["us-east-1"]'

# Remove all instances (one at a time)
for key in "us-east-1" "us-west-2" "eu-west-1"; do
  tofu state rm "aws_s3_bucket.regional[\"$key\"]"
done
```

## Dry Run with -dry-run

```bash
# Preview what would be removed without making changes
tofu state rm -dry-run aws_instance.web module.old_module

# Output:
# Would remove aws_instance.web
# Would remove module.old_module.aws_vpc.main
# Would remove module.old_module.aws_subnet.public
```

## Practical Scenarios

### Abandoning Manual Resources

```bash
# Scenario: someone manually created an instance that's in state
# You want to "forget" it and manage it manually going forward

# Check it exists in state
tofu state list | grep aws_instance.manual_server

# Remove from state (instance still runs in AWS)
tofu state rm aws_instance.manual_server

# Remove from config
# (delete the resource block from your .tf files)

# Verify plan is clean
tofu plan  # Should show no changes
```

### Migrating to a New Workspace

```bash
# Scenario: split a monolith into separate workspaces

# Workspace 1 (networking): import the VPC
tofu workspace select networking
tofu import aws_vpc.main vpc-0abc12345

# Workspace 2 (app): remove the VPC so it's no longer managed here
tofu workspace select app
tofu state rm aws_vpc.main

# Now the VPC is managed by workspace 1, not workspace 2
```

### Removing a Deleted Resource from State

```bash
# Scenario: someone deleted an instance directly in AWS
# Now tofu plan shows an error refreshing it

# Remove the deleted resource from state
tofu state rm aws_instance.accidentally_deleted

# Plan now shows it as "to add" (if you want to recreate)
# or remove from config if you don't want it
tofu plan
```

### Bulk Removal Script

```bash
#!/bin/bash
# remove-from-state.sh

RESOURCES_TO_REMOVE=(
  "aws_instance.deprecated_web"
  "aws_security_group.deprecated_web"
  "aws_eip.deprecated_web"
)

for resource in "${RESOURCES_TO_REMOVE[@]}"; do
  echo "Removing: $resource"
  tofu state rm "$resource"
done

echo "Done. Run 'tofu plan' to verify."
```

## Backup and Safety

```bash
# state rm automatically creates a backup at terraform.tfstate.backup
# But for safety, create your own:
cp terraform.tfstate terraform.tfstate.pre-rm

# Use -backup flag for custom backup location
tofu state rm -backup=before-removal.tfstate aws_instance.web

# To undo a state rm (restore from backup):
cp terraform.tfstate.backup terraform.tfstate
```

## After Removing from State

Once removed from state, if the resource definition still exists in config, the next `tofu plan` will show it as "to add":

```bash
tofu state rm aws_instance.web

# If resource "aws_instance" "web" {} is still in config:
tofu plan
# Plan: 1 to add, 0 to change, 0 to destroy
# aws_instance.web will be created

# To avoid recreation, also remove from config
# Or re-import: tofu import aws_instance.web i-0123456789
```

## Using removed Block Instead

For code-tracked removal, use the `removed` block (OpenTofu 1.7+):

```hcl
removed {
  from = aws_instance.web

  lifecycle {
    destroy = false  # Remove from state without destroying
  }
}
```

## Conclusion

`tofu state rm` is a surgical tool for removing resources from OpenTofu management without touching the real infrastructure. Use it to abandon management of resources, fix broken states, or migrate resources between workspaces. Always dry-run first, create backups before bulk operations, and prefer the `removed` block for team environments where changes should be tracked in code.
