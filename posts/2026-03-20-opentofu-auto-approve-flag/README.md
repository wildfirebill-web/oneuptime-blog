# How to Use the -auto-approve Flag in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to use the -auto-approve flag in OpenTofu to skip interactive confirmation prompts for automated deployments.

## Introduction

By default, `tofu apply` and `tofu destroy` show a plan and ask for confirmation before making changes. The `-auto-approve` flag skips this prompt, allowing OpenTofu to proceed immediately without user input. It is designed for automated CI/CD pipelines where human confirmation is handled at a higher level (PR review, deployment approvals, etc.).

## Basic Usage

```bash
# Apply without confirmation prompt

tofu apply -auto-approve

# Destroy without confirmation prompt
tofu destroy -auto-approve
```

## What It Skips

```bash
# Without -auto-approve
tofu apply
# Plan: 3 to add, 1 to change, 0 to destroy.
#
# Do you want to perform these actions?
#   OpenTofu will perform the actions described above.
#   Only 'yes' will be accepted to approve.
#
# Enter a value:  ← PROMPT - requires human input

# With -auto-approve
tofu apply -auto-approve
# Plan: 3 to add, 1 to change, 0 to destroy.
# (Immediately proceeds without prompting)
```

## With Saved Plan Files

```bash
# Applying a saved plan file NEVER prompts for confirmation
tofu plan -out=tfplan
tofu apply tfplan    # No -auto-approve needed - plan is already approved
```

This is the recommended pattern: use plan files so the apply step is implicitly approved by the plan review step.

## CI/CD Usage

```bash
# GitHub Actions example
- name: Apply
  run: tofu apply -auto-approve -var-file=environments/${{ env.ENVIRONMENT }}.tfvars
```

## With Other Flags

```bash
# Combine with other flags
tofu apply \
  -auto-approve \
  -parallelism=20 \
  -var-file=production.tfvars
```

## Environment Variable Alternative

```bash
# Set via environment variable
export TF_CLI_ARGS_apply="-auto-approve"
tofu apply  # Same as tofu apply -auto-approve
```

## When NOT to Use -auto-approve

```bash
# Local development: always review before applying
tofu plan   # Review first
tofu apply  # Interactive confirmation is a safety check

# Production: prefer saved plan files over -auto-approve
tofu plan -out=tfplan   # Plan and review
tofu apply tfplan        # Apply exactly what was reviewed
```

## Safety Patterns for Automated Pipelines

**Pattern 1: Require manual approval in GitHub Actions:**
```yaml
jobs:
  apply:
    environment: production  # Requires manual approval configured in repo settings
    steps:
      - run: tofu apply -auto-approve
```

**Pattern 2: Apply only saved plans:**
```yaml
- name: Plan
  run: tofu plan -out=tfplan

- name: Apply (auto-approved because plan was reviewed)
  run: tofu apply tfplan
  # No -auto-approve needed with plan file
```

**Pattern 3: Restrict to non-destructive changes only:**
```bash
# Fail if plan includes destroy operations
DESTROY_COUNT=$(tofu show -json tfplan | jq '[.resource_changes[] | select(.change.actions[] == "delete")] | length')
if [ "$DESTROY_COUNT" -gt 0 ]; then
  echo "Error: Plan includes $DESTROY_COUNT resource deletions - manual review required"
  exit 1
fi
tofu apply tfplan
```

## Conclusion

`-auto-approve` is for automated pipelines only - never use it for local development where reviewing the plan before applying is a critical safety step. The preferred pattern in CI/CD is to save the plan with `tofu plan -out=tfplan`, have it reviewed (in a PR or approval gate), and then apply with `tofu apply tfplan` - the plan file itself serves as the approval mechanism. Reserve `-auto-approve` for non-production automated deployments where a human review occurred at the PR/merge level.
