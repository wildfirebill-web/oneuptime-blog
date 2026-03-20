# How to Use Partial Backend Configuration in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backends

Description: Learn how to use partial backend configuration in OpenTofu to separate static backend settings from dynamic credentials and environment-specific values.

## Introduction

Partial backend configuration lets you define some backend settings in code and supply the rest at `tofu init` time. This keeps credentials and environment-specific values out of source control while still specifying the backend type and non-sensitive settings in version-controlled files.

## Why Use Partial Configuration

Backend blocks cannot reference variables or locals. Partial configuration solves this by letting you:
- Store the backend type and non-sensitive settings in code
- Pass credentials, bucket names, or environment-specific paths at init time
- Use different values per environment without duplicating code

## The Pattern

```hcl
# backend.tf — committed to source control
terraform {
  backend "s3" {
    region = "us-east-1"
    # bucket and key are passed at init time
  }
}
```

```bash
# init time — not committed
tofu init \
  -backend-config="bucket=acme-tofu-state-production" \
  -backend-config="key=infrastructure/terraform.tfstate" \
  -backend-config="dynamodb_table=terraform-lock"
```

## Using a Backend Config File

Store the dynamic values in a file that is gitignored:

```hcl
# backends/production.hcl — gitignored
bucket         = "acme-tofu-state-production"
key            = "infrastructure/terraform.tfstate"
dynamodb_table = "terraform-lock"
encrypt        = true
```

```bash
tofu init -backend-config=backends/production.hcl
```

```bash
# staging
tofu init -backend-config=backends/staging.hcl
```

## Multiple Backend Config Files by Environment

```
backends/
├── production.hcl    # gitignored
├── staging.hcl       # gitignored
└── development.hcl   # gitignored

backend.tf            # committed
```

```hcl
# backend.tf
terraform {
  backend "s3" {
    region = "us-east-1"
    # All other values supplied at init
  }
}
```

## Mixing File and Flag Configuration

You can combine a config file with additional `-backend-config` flags:

```bash
tofu init \
  -backend-config=backends/base.hcl \
  -backend-config="dynamodb_table=terraform-lock-prod"
```

## GCS Partial Configuration Example

```hcl
# backend.tf
terraform {
  backend "gcs" {
    # prefix supplied at init
  }
}
```

```bash
tofu init \
  -backend-config="bucket=acme-tofu-state" \
  -backend-config="prefix=production/infrastructure"
```

## Azure Partial Configuration Example

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "acmetofustate"
    container_name       = "tfstate"
    # key supplied at init
  }
}
```

```bash
tofu init -backend-config="key=production.terraform.tfstate"
```

## CI/CD Integration

```yaml
# GitHub Actions
- name: Init
  run: |
    tofu init \
      -backend-config="bucket=${{ vars.STATE_BUCKET }}" \
      -backend-config="key=${{ vars.ENVIRONMENT }}/terraform.tfstate" \
      -backend-config="dynamodb_table=${{ vars.LOCK_TABLE }}"
  env:
    AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}
```

## Reconfiguring the Backend

If backend settings change between inits, use `-reconfigure`:

```bash
tofu init -reconfigure -backend-config=backends/new-config.hcl
```

Or `-migrate-state` to copy existing state to the new location:

```bash
tofu init -migrate-state -backend-config=backends/new-config.hcl
```

## Conclusion

Partial backend configuration is the recommended approach for keeping credentials and environment-specific values out of source control. Use backend config files per environment (gitignored) and pass them at init time. For CI/CD, inject values via environment variables or secrets manager and pass them as `-backend-config` flags during the init step.
