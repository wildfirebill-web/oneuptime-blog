# How to Migrate from Terraform Enterprise to OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform Enterprise, Migration, Infrastructure as Code, HCP Terraform

Description: Learn how to migrate your infrastructure configurations and workflows from Terraform Enterprise to OpenTofu, including state migration, provider updates, and CI/CD changes.

## Introduction

With HashiCorp's license change from MPL to BSL, many teams are evaluating OpenTofu as an open-source alternative to Terraform Enterprise. OpenTofu is a fork of Terraform that remains open-source and maintains compatibility with existing configurations.

## Pre-Migration Assessment

Before migrating, inventory your current usage:

```bash
# List all workspaces

terraform workspace list

# Check provider versions in use
grep -r "required_providers" . --include="*.tf"

# Check for any Enterprise-specific features
grep -r "tfe_" . --include="*.tf"
```

## Installing OpenTofu

```bash
# macOS
brew install opentofu

# Linux (official install script)
curl --proto '=https' --tlsv1.2 -fsSL https://get.opentofu.org/install-opentofu.sh | sh

# Verify
tofu version
```

## Migrating State

### Option 1: Using Existing State Files

If you have access to state files, initialize OpenTofu with the same backend:

```bash
tofu init
tofu plan  # verify no unexpected changes
```

### Option 2: Migrating from TFE/HCP Terraform

Export state from Terraform Enterprise:

```bash
# Get workspace ID from TFE API
curl -H "Authorization: Bearer $TFE_TOKEN" \
  https://app.terraform.io/api/v2/organizations/my-org/workspaces/my-workspace

# Pull state
terraform state pull > terraform.tfstate
```

Configure a new backend (e.g., S3):

```hcl
terraform {
  backend "s3" {
    bucket = "my-tofu-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}
```

Push state to new backend:

```bash
tofu init
tofu state push terraform.tfstate
```

## Updating Provider Sources

OpenTofu uses the same provider registry but ensure sources are explicit:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

## Replacing TFE-Specific Resources

If you used the `tfe` provider for workspace management, replace with OpenTofu equivalents or direct configuration.

## Updating CI/CD Pipelines

Replace `terraform` commands with `tofu`:

```yaml
# Before
- run: terraform init && terraform apply -auto-approve

# After
- run: tofu init && tofu apply -auto-approve
```

## Validation

```bash
tofu init
tofu validate
tofu plan
```

Review the plan carefully - it should show no changes if migration was successful.

## Conclusion

Migrating from Terraform Enterprise to OpenTofu is straightforward for most configurations. The primary work involves state migration, updating CI/CD pipelines, and replacing any TFE provider resources. OpenTofu's binary compatibility with Terraform configurations means most migrations complete without code changes.
