# How to Save and Apply Plan Files in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps

Description: Learn how to save OpenTofu plan files with -out, review them, and apply them exactly to ensure what you reviewed is what gets deployed.

## Introduction

Saving plan files (`tofu plan -out`) and applying them later (`tofu apply planfile`) is one of the most important safety practices in OpenTofu. It ensures that the exact changes you reviewed are the changes that get applied — no drift can occur between the review and deployment.

## Saving a Plan

```bash
# Save plan to a file
tofu plan -out=changes.tfplan

# With variables
tofu plan -var-file=production.tfvars -out=prod-changes.tfplan

# Workspace-specific
tofu workspace select production
tofu plan -var-file=production.tfvars -out=production-$(date +%Y%m%d).tfplan
```

## Reviewing a Saved Plan

```bash
# Human-readable review
tofu show changes.tfplan

# JSON for automated analysis
tofu show -json changes.tfplan

# Extract key metrics
tofu show -json changes.tfplan | jq '
  .resource_changes |
  group_by(.change.actions[0]) |
  map({action: .[0].change.actions[0], count: length})
'
```

## Applying a Saved Plan

```bash
# Apply the saved plan exactly
tofu apply changes.tfplan

# No confirmation prompt needed for saved plans
# (the review is the confirmation)
```

## Plan File Format

Saved plan files are binary (not plain text JSON) and are encrypted if you've configured state encryption:

```bash
# Don't try to cat or read plan files directly
file changes.tfplan
# changes.tfplan: data  (binary format)

# Use tofu show to read them
tofu show changes.tfplan
```

## Plan Validity Window

A saved plan is valid as long as:
1. No changes have been made to the configuration since planning
2. No resources have changed state since planning
3. The backend/state hasn't changed

If state changes after planning:

```bash
# Plan becomes stale if state changes
tofu apply outdated.tfplan
# Error: Applied changes since plan was created...
# You need to re-plan
```

## CI/CD Workflow with Plan Files

```yaml
# GitHub Actions: Plan → Review → Apply
jobs:
  plan:
    runs-on: ubuntu-latest
    outputs:
      has-changes: ${{ steps.check.outputs.has-changes }}
    steps:
      - uses: actions/checkout@v4

      - name: Plan
        run: |
          tofu plan -out=plan.tfplan -detailed-exitcode
        id: plan
        continue-on-error: true

      - name: Check for Changes
        id: check
        run: |
          echo "has-changes=${{ steps.plan.outcome == 'failure' || steps.plan.outputs.exitcode == '2' }}" >> $GITHUB_OUTPUT

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: tfplan-${{ github.sha }}
          path: plan.tfplan
          retention-days: 7

  apply:
    needs: plan
    if: needs.plan.outputs.has-changes == 'true' && github.ref == 'refs/heads/main'
    environment: production  # Requires approval
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: tfplan-${{ github.sha }}

      - name: Apply
        run: tofu apply plan.tfplan
```

## Naming Plan Files

Use descriptive, timestamped names:

```bash
# Include timestamp and environment
tofu plan -out="prod-$(date +%Y%m%d-%H%M%S).tfplan"

# Include PR or commit reference
tofu plan -out="pr-123-changes.tfplan"

# Include workspace
tofu plan -out="${TF_WORKSPACE}-$(date +%Y%m%d).tfplan"
```

## Conclusion

Saving and applying plan files is the gold standard for safe infrastructure deployments. The two-step workflow (plan → review → apply plan file) ensures that what you approved is exactly what gets deployed, with no possibility of drift between review and execution. Implement this pattern in all production CI/CD pipelines for maximum safety and auditability.
