# How to Update Documentation When Migrating to OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Documentation, Migration, Infrastructure as Code, DevOps

Description: Learn what documentation to update when migrating from Terraform to OpenTofu - runbooks, READMEs, module docs, onboarding guides, and internal wikis.

## Introduction

Documentation updates are often overlooked during tool migrations. When migrating from Terraform to OpenTofu, outdated documentation causes confusion and slows down new team members. This guide covers a systematic approach to updating all relevant docs.

## Files to Update

### 1. Module README.md

Every Terraform module README needs updates:

```markdown
<!-- Before -->
## Requirements

| Name | Version |
|------|---------|
| terraform | >= 1.5 |
| aws | >= 5.0 |

## Usage

\```hcl
module "vpc" {
  source  = "my-org/vpc/aws"
  version = "~> 2.0"
}
\```

Run with:
\```bash
terraform init
terraform plan
terraform apply
\```
```

```markdown
<!-- After -->
## Requirements

| Name | Version |
|------|---------|
| opentofu | >= 1.8 |
| aws | >= 5.0 |

## Usage

\```hcl
module "vpc" {
  source  = "my-org/vpc/aws"
  version = "~> 2.0"
}
\```

Run with:
\```bash
tofu init
tofu plan
tofu apply
\```

> **Note**: This module is compatible with both OpenTofu >= 1.8 and Terraform >= 1.5.
```

### 2. CONTRIBUTING.md

```markdown
## Development Setup

### Prerequisites

Install OpenTofu:
\```bash
# macOS

brew install opentofu

# Linux
curl -fsSL https://get.opentofu.org/install-opentofu.sh | sh
\```

### Running Tests

\```bash
# Unit tests
tofu validate

# Integration tests (requires AWS credentials)
cd test/
go test -v -run TestVPCModule -timeout 30m
\```

### Making Changes

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Run `tofu validate` and `tflint`
5. Submit a pull request
```

### 3. Operations Runbook

```markdown
# Infrastructure Operations Runbook

## Tool Version
OpenTofu v1.9.x (pinned in `.tool-versions`)

## Daily Operations

### Check for Drift
\```bash
tofu -chdir=environments/production init
tofu -chdir=environments/production plan -refresh-only
\```

### Deploy Change
\```bash
# 1. Create feature branch and make changes
# 2. Open PR - CI will post plan as comment
# 3. Review plan, get approval
# 4. Merge - CI applies automatically
\```

### Emergency Apply
If CI is unavailable, apply manually:
\```bash
tofu -chdir=environments/production init
tofu -chdir=environments/production plan -out=emergency.tfplan
# Review plan carefully
tofu -chdir=environments/production apply emergency.tfplan
\```

### State Operations
\```bash
# List resources in state
tofu state list

# Show specific resource
tofu state show aws_instance.app

# Move resource
tofu state mv aws_instance.old_name aws_instance.new_name

# Remove resource from state (without destroying)
tofu state rm aws_instance.orphaned
\```
```

### 4. Onboarding Documentation

```markdown
# Infrastructure Onboarding Guide

## Tools You'll Need

1. **OpenTofu** - Infrastructure as Code tool
   \```bash
   brew install opentofu   # macOS
   tofu --version          # Verify: OpenTofu v1.9.x
   \```

2. **tflint** - OpenTofu linter
   \```bash
   brew install tflint
   \```

3. **AWS CLI** - For AWS credentials
   \```bash
   brew install awscli
   aws configure sso   # Set up SSO login
   \```

## Your First Week

- [ ] Complete OpenTofu fundamentals (link to internal course)
- [ ] Shadow a senior engineer on a tofu plan review
- [ ] Make a small change in a dev environment
- [ ] Review the CI/CD pipeline documentation

## Key Concepts

- **State**: OpenTofu tracks what exists in state files stored in S3
- **Plan**: Always run `tofu plan` before `tofu apply`
- **Modules**: Reusable infrastructure components in `modules/`
- **Environments**: Separate configs in `environments/dev`, `staging`, `production`
```

### 5. Update CI/CD Pipeline Documentation

```markdown
# CI/CD Pipeline Guide

## OpenTofu GitHub Actions Pipeline

The pipeline runs on every PR and push to main:

| Event | Action |
|-------|--------|
| PR opened/updated | `tofu plan`, post as PR comment |
| PR merged to main | `tofu apply` |
| Nightly | Drift detection (`tofu plan -refresh-only`) |

## Pipeline Files
- `.github/workflows/opentofu-plan.yml` - PR planning
- `.github/workflows/opentofu-apply.yml` - Deployment
- `.github/workflows/drift-detection.yml` - Nightly drift check

## Required Secrets
| Secret | Description |
|--------|-------------|
| `AWS_ROLE_ARN` | IAM role for GitHub Actions OIDC |
```

## Automated Documentation Check

```bash
#!/bin/bash
# scripts/check-docs.sh - find remaining "terraform" references in docs

echo "Checking for outdated 'terraform' CLI references in documentation..."

# Find files with terraform command references (not provider names)
grep -rn "terraform init\|terraform plan\|terraform apply\|terraform destroy\|terraform state" \
  --include="*.md" \
  . | grep -v ".terraform" | grep -v "terraform.tfstate"

echo "Review the above references and update to 'tofu' where appropriate."
```

## Conclusion

Systematic documentation updates prevent migration confusion. Update module READMEs, the CONTRIBUTING guide, operations runbooks, and onboarding documentation. Run a grep for `terraform init|plan|apply` in markdown files to find remaining references. The automated check script makes it easy to verify documentation completeness before declaring the migration done.
