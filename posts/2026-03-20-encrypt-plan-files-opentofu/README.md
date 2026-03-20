# How to Encrypt Plan Files in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Security, Encryption

Description: Learn how to encrypt OpenTofu plan files to protect sensitive infrastructure details and prevent plan tampering in CI/CD workflows.

## Introduction

OpenTofu plan files contain detailed information about your infrastructure, including resource attributes with sensitive values. When plan files are stored in CI/CD artifacts, passed between pipeline stages, or stored for audit purposes, encrypting them prevents unauthorized access and tampering. OpenTofu 1.7+ supports native plan file encryption.

## Why Encrypt Plan Files?

Plan files may contain:
- Database connection strings from data sources
- Resource attributes with sensitive values
- Configuration details that reveal your architecture
- Potential attack vectors if tampered before apply

## Step 1: Configure Plan File Encryption

Add the `plan` block to your encryption configuration:

```hcl
# encryption.tf
terraform {
  required_version = ">= 1.7.0"

  encryption {
    key_provider "pbkdf2" "key" {
      passphrase = var.encryption_passphrase
    }

    method "aes_gcm" "method" {
      keys = key_provider.pbkdf2.key
    }

    # Encrypt state files
    state {
      method   = method.aes_gcm.method
      enforced = true
    }

    # Also encrypt plan files
    plan {
      method   = method.aes_gcm.method
      enforced = true
    }
  }
}
```

## Step 2: Save an Encrypted Plan

```bash
# Save an encrypted plan file
tofu plan -out=infrastructure.tfplan

# The plan file is now encrypted
file infrastructure.tfplan
# Not readable as plain text
```

## Step 3: Store the Plan File Securely

In CI/CD pipelines, store the encrypted plan file in a secure artifact store:

```yaml
# GitHub Actions example
jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Plan
        env:
          TF_VAR_encryption_passphrase: ${{ secrets.STATE_ENCRYPTION_PASSPHRASE }}
        run: |
          tofu init
          tofu plan -out=infrastructure.tfplan

      - name: Upload Plan
        uses: actions/upload-artifact@v4
        with:
          name: terraform-plan
          path: infrastructure.tfplan
          retention-days: 30

  apply:
    needs: plan
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Download Plan
        uses: actions/download-artifact@v4
        with:
          name: terraform-plan

      - name: Apply
        env:
          TF_VAR_encryption_passphrase: ${{ secrets.STATE_ENCRYPTION_PASSPHRASE }}
        run: tofu apply infrastructure.tfplan
```

## Step 4: Apply an Encrypted Plan

Applying an encrypted plan requires the same encryption configuration:

```bash
# Apply requires the same key that was used to create the plan
export TF_VAR_encryption_passphrase="your-passphrase"

tofu apply infrastructure.tfplan

# If the plan was encrypted with a different key, apply fails:
# Error: Failed to decrypt plan file
```

## Using Different Keys for Plan and State

You can use separate keys for plan and state files:

```hcl
terraform {
  encryption {
    # Key for state
    key_provider "aws_kms" "state_key" {
      kms_key_id = "alias/terraform-state"
      region     = "us-east-1"
    }

    # Separate key for plans
    key_provider "aws_kms" "plan_key" {
      kms_key_id = "alias/terraform-plans"
      region     = "us-east-1"
    }

    method "aes_gcm" "state_method" {
      keys = key_provider.aws_kms.state_key
    }

    method "aes_gcm" "plan_method" {
      keys = key_provider.aws_kms.plan_key
    }

    state {
      method   = method.aes_gcm.state_method
      enforced = true
    }

    plan {
      method   = method.aes_gcm.plan_method
      enforced = true
    }
  }
}
```

## Viewing Plan Contents Securely

To review an encrypted plan:

```bash
# Use tofu show (requires the encryption key to be configured)
tofu show infrastructure.tfplan

# Or for JSON output
tofu show -json infrastructure.tfplan | jq '.resource_changes[]'

# Get a human-readable diff
tofu show infrastructure.tfplan | grep -E "^[[:space:]]*(+|-|~)"
```

## Plan File Integrity

Encrypted plan files include an integrity check — if the plan file is tampered with, apply will fail:

```bash
# Tampering with the plan file is detected
echo "tampered" >> infrastructure.tfplan
tofu apply infrastructure.tfplan
# Error: plan file integrity check failed — the file has been modified
```

## Conclusion

Encrypting OpenTofu plan files adds an important layer of security to your CI/CD workflow. It prevents sensitive data leakage through artifact stores, ensures plan integrity (tampering is detected), and creates a secure hand-off between plan and apply stages. Configure plan encryption alongside state encryption for comprehensive protection of all OpenTofu artifacts.
