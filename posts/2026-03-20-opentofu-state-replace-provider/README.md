---
title: "Using tofu state replace-provider in OpenTofu"
author: nawazdhandala
tags: opentofu, terraform, iac, state, providers
description: "Learn how to use tofu state replace-provider to update provider references in state when migrating between provider sources."
---

# Using tofu state replace-provider in OpenTofu

The `tofu state replace-provider` command updates the provider source addresses stored in state. It's essential when migrating from one provider source to another — for example, moving from a community provider fork to an official release, or switching registry hosts.

## When to Use replace-provider

- **Provider renamed**: A provider's registry namespace changed
- **Registry migration**: Moving from Terraform Registry to OpenTofu Registry
- **Fork adoption**: Switching from a community fork to official provider
- **Custom provider**: Moving from local to published registry source
- **Organization change**: Provider moved from one org to another in registry

## Basic Syntax

```bash
tofu state replace-provider OLD_PROVIDER_SOURCE NEW_PROVIDER_SOURCE
```

## Common Migration: Terraform to OpenTofu

When migrating from Terraform to OpenTofu, some provider sources may need updating:

```bash
# Migrate from Terraform registry to OpenTofu registry
tofu state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws

# Check what providers are currently in state
tofu providers
```

## Migrating a Renamed Provider

```bash
# Example: provider moved from old namespace to new
tofu state replace-provider \
  registry.opentofu.org/old-org/myprovider \
  registry.opentofu.org/new-org/myprovider
```

## Migrating from a Fork

```bash
# Switch from community fork to official provider
tofu state replace-provider \
  registry.opentofu.org/community-org/aws-fork \
  registry.opentofu.org/hashicorp/aws
```

## Dry Run

```bash
# Preview changes without modifying state
tofu state replace-provider \
  -dry-run \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws

# Output:
# Showing planned state changes:
# replace provider "registry.terraform.io/hashicorp/aws" 
# with "registry.opentofu.org/hashicorp/aws"
# Affected resources:
#   aws_instance.web
#   aws_vpc.main
#   module.networking.aws_subnet.public[0]
#   ...
```

## Complete Migration Workflow

```bash
# Step 1: Check current provider in state
tofu providers

# Step 2: Update your configuration
# Change required_providers source in versions.tf
# Before:
# aws = {
#   source = "registry.terraform.io/hashicorp/aws"
# }
# After:
# aws = {
#   source = "registry.opentofu.org/hashicorp/aws"
# }

# Step 3: Run replace-provider
tofu state replace-provider \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws

# Step 4: Reinitialize to download from new source
tofu init -upgrade

# Step 5: Verify plan shows no unexpected changes
tofu plan
```

## Replacing Multiple Providers

```bash
# Migrate all hashicorp providers at once
PROVIDERS=(aws google azurerm kubernetes random null)

for provider in "${PROVIDERS[@]}"; do
  tofu state replace-provider \
    "registry.terraform.io/hashicorp/${provider}" \
    "registry.opentofu.org/hashicorp/${provider}" 2>/dev/null || true
done
```

## Checking State Providers

```bash
# See all providers referenced in state
tofu providers

# More detailed view
tofu show -json | jq '.values.root_module | .. | .provider_name? | select(. != null)' | sort -u
```

## Auto-Approve Flag

```bash
# Skip confirmation prompt (useful in automation)
tofu state replace-provider \
  -auto-approve \
  registry.terraform.io/hashicorp/aws \
  registry.opentofu.org/hashicorp/aws
```

## Backup Handling

```bash
# replace-provider creates a backup automatically
# Verify backup exists before proceeding
ls -la terraform.tfstate.backup

# For extra safety, create your own backup
cp terraform.tfstate "terraform.tfstate.before-replace-$(date +%Y%m%d%H%M%S)"

# Run the replacement
tofu state replace-provider \
  old-source/provider \
  new-source/provider
```

## Verifying Success

```bash
# After replacement, check the state
cat terraform.tfstate | grep '"provider"' | head -5
# Should show the new provider source

# Run init to install the new provider
tofu init

# Plan to verify no unexpected changes
tofu plan
# Expected: No changes. Infrastructure is up-to-date.
```

## Conclusion

`tofu state replace-provider` is the correct tool for migrating between provider sources without affecting your infrastructure. Use it when switching from Terraform to OpenTofu registries, adopting official providers over forks, or when providers move between registry namespaces. Always dry-run first, back up your state, and verify with a clean plan after the migration.
