# How to Save and Apply Plan Files in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, CLI

Description: Learn how to save OpenTofu plans to files and apply them later to guarantee that exactly what was reviewed gets deployed.

## Introduction

OpenTofu can save a plan to a binary file with `tofu plan -out=<file>`. Applying a saved plan file guarantees that exactly what was reviewed is what gets applied — no risk of new changes appearing between plan and apply. This is the recommended pattern for production CI/CD pipelines.

## Saving a Plan

```bash
# Save plan to a file
tofu plan -out=tfplan

# With variables
tofu plan -out=tfplan -var-file=environments/production.tfvars
```

## Applying a Saved Plan

```bash
# Apply the exact saved plan
tofu apply tfplan

# No confirmation prompt — the plan file IS the approval
# No -auto-approve needed
```

## Reviewing a Saved Plan

```bash
# Human-readable review of the plan
tofu show tfplan

# JSON review for tooling
tofu show -json tfplan
```

## The Full Plan-Review-Apply Workflow

```bash
# Step 1: Plan and save
tofu plan -out=tfplan

# Step 2: Review (human or automated)
tofu show tfplan

# Step 3: Apply (exactly what was reviewed)
tofu apply tfplan
```

## Plan Files Are Binary

Plan files are binary — do not try to edit them or view them with a text editor. Use `tofu show` to read them.

```bash
# Do NOT do this
cat tfplan | jq .  # Won't work — not a JSON file

# DO this
tofu show -json tfplan | jq .  # Works correctly
```

## Encrypted Plan Files

In OpenTofu, plan files respect state encryption settings:

```hcl
# encryption.tf
terraform {
  encryption {
    key_provider "pbkdf2" "main" {
      passphrase = var.encryption_passphrase
    }
    method "aes_gcm" "main" {
      keys = key_provider.pbkdf2.main
    }
    plan {
      method = method.aes_gcm.main
    }
  }
}
```

```bash
# Plan file is encrypted automatically
tofu plan -out=tfplan
# Cannot be read without the encryption key
```

## CI/CD Pattern: Plan in PR, Apply on Merge

```yaml
# .github/workflows/plan.yml
on: [pull_request]
jobs:
  plan:
    steps:
      - run: tofu init -input=false
      - run: tofu plan -out=tfplan -input=false
      - uses: actions/upload-artifact@v4
        with:
          name: tfplan
          path: tfplan
```

```yaml
# .github/workflows/apply.yml
on:
  push:
    branches: [main]
jobs:
  apply:
    environment: production  # Manual approval gate
    steps:
      - uses: actions/download-artifact@v4
        with: { name: tfplan }
      - run: tofu apply tfplan
```

## Security Considerations

Plan files contain the full state snapshot and all planned changes, including sensitive values. Treat them as secrets:

```bash
# In GitHub Actions: use encrypted artifacts or pass through secrets
# Do NOT commit plan files to version control
echo "tfplan" >> .gitignore
echo "*.tfplan" >> .gitignore
```

## Plan File Expiry

Plan files are valid only as long as the state hasn't changed. If someone else applies changes between your plan and apply, OpenTofu detects the state mismatch and errors:

```bash
tofu apply tfplan
# Error: Saved plan is stale
# The plan was created against a different version of the state.
# Run tofu plan again to create a fresh plan.
```

## Conclusion

Saved plan files are the foundation of safe automated deployments. They decouple the plan step (where changes are reviewed) from the apply step (where changes execute), and they guarantee that exactly what was reviewed is what gets applied. Always use plan files in production pipelines, treat them as sensitive artifacts, and re-plan if state changes between plan and apply.
