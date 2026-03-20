# How to Use the OpenTofu CLI Quick Reference

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, CLI, Quick Reference, Commands, Infrastructure as Code

Description: A comprehensive quick reference for all commonly used OpenTofu CLI commands with flags and usage examples.

## Introduction

This quick reference covers the most frequently used OpenTofu CLI commands. Keep this as a cheat sheet for day-to-day operations.

## Initialization

```bash
# Initialize working directory (download providers and modules)
tofu init

# Initialize and upgrade providers/modules to latest allowed versions
tofu init -upgrade

# Initialize with a specific backend config
tofu init -backend-config="bucket=my-state-bucket"

# Initialize without downloading modules
tofu init -get=false
```

## Planning

```bash
# Show what changes will be made
tofu plan

# Save plan to file (use with apply for consistency)
tofu plan -out=plan.tfplan

# Plan with specific variable values
tofu plan -var="region=us-west-2" -var-file="prod.tfvars"

# Plan only specific resources
tofu plan -target=aws_s3_bucket.main

# Plan excluding specific resources
tofu plan -exclude=aws_db_instance.main

# Plan a destroy operation
tofu plan -destroy

# Plan with detailed exit code for CI/CD
tofu plan -detailed-exitcode
# Exit 0: no changes, Exit 1: error, Exit 2: changes present

# Refresh state only (no changes)
tofu apply -refresh-only
```

## Applying

```bash
# Apply with interactive confirmation
tofu apply

# Apply without confirmation (CI/CD)
tofu apply -auto-approve

# Apply a saved plan file
tofu apply plan.tfplan

# Apply with variable values
tofu apply -var="environment=prod"

# Force replacement of a specific resource
tofu apply -replace=aws_instance.web

# Limit parallel operations
tofu apply -parallelism=5
```

## Destroying

```bash
# Destroy all resources (interactive confirmation)
tofu destroy

# Destroy without confirmation
tofu destroy -auto-approve

# Destroy only specific resources
tofu destroy -target=module.dev_environment
```

## State Operations

```bash
tofu state list                          # list all resources
tofu state show aws_s3_bucket.main       # show resource details
tofu state mv old_address new_address    # move/rename resource
tofu state rm aws_s3_bucket.temp         # remove from state
tofu state pull > backup.tfstate         # export state
tofu state push terraform.tfstate        # import state
```

## Workspace Operations

```bash
tofu workspace list              # list workspaces
tofu workspace new staging       # create workspace
tofu workspace select staging    # switch workspace
tofu workspace show              # current workspace
tofu workspace delete staging    # delete workspace
```

## Output and Inspection

```bash
tofu output                      # show all outputs
tofu output vpc_id               # show specific output
tofu output -json                # JSON format for scripts
tofu show                        # show current state
tofu show plan.tfplan            # show saved plan
tofu graph                       # dependency graph (DOT format)
tofu version                     # show OpenTofu version
tofu providers                   # list required providers
```

## Validation and Formatting

```bash
tofu validate             # validate configuration syntax
tofu fmt                  # format HCL files
tofu fmt -check           # check formatting (CI/CD)
tofu fmt -recursive       # format all subdirectories
```

## Interactive Console

```bash
tofu console                     # open interactive REPL
# In console:
# > var.environment
# "production"
# > aws_s3_bucket.main.arn
# "arn:aws:s3:::my-bucket"
```

## Summary

These commands cover the full OpenTofu workflow from initialization through destruction. For daily use, the core commands are `tofu init`, `tofu plan -out=plan.tfplan`, `tofu show plan.tfplan`, and `tofu apply plan.tfplan`. Use `tofu fmt` and `tofu validate` in pre-commit hooks to maintain code quality automatically.
