# How to Upgrade OpenTofu from 1.10 to 1.11

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Upgrade, Version Migration, Infrastructure as Code, DevOps

Description: A comprehensive guide to upgrading OpenTofu from version 1.10 to 1.11, including pre-upgrade checks, migration steps, and post-upgrade validation.

## Introduction

OpenTofu 1.11 brings continued improvements to the open-source infrastructure tooling ecosystem. This guide provides a systematic approach to upgrading from 1.10 to 1.11 safely.

## Understanding the Release Notes

Before upgrading, always check the official release notes:

```bash
# View release notes via GitHub API

curl -s https://api.github.com/repos/opentofu/opentofu/releases | \
  jq '.[] | select(.tag_name | startswith("v1.11")) | {tag: .tag_name, body: .body}' | head -100

# Or visit: https://github.com/opentofu/opentofu/releases
```

## Pre-Upgrade Checklist

```bash
# Step 1: Verify current version
tofu version
# OpenTofu v1.10.x

# Step 2: Check provider lock file
cat .terraform.lock.hcl | head -30

# Step 3: Backup everything
cp terraform.tfstate terraform.tfstate.backup-pre-1.11
cp terraform.tfstate.backup terraform.tfstate.backup.backup-pre-1.11 2>/dev/null || true

# Step 4: Stash any uncommitted work
git stash

# Step 5: Create upgrade branch
git checkout -b feature/upgrade-opentofu-1.11

# Step 6: Record baseline plan
tofu plan -out=pre-upgrade-plan.tfplan
```

## Install OpenTofu 1.11

```bash
# Via tofuenv (recommended)
tofuenv install latest:^1.11
tofuenv use 1.11.0
tofu version

# Via package manager
sudo apt-get update && sudo apt-get upgrade tofu

# Via binary
TOFU_VERSION="1.11.0"
curl -fsSL \
  "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip" \
  -o "tofu-${TOFU_VERSION}.zip"

# Verify checksum
curl -fsSL \
  "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_SHA256SUMS" \
  -o "SHA256SUMS"
sha256sum -c --ignore-missing SHA256SUMS

# Install
unzip "tofu-${TOFU_VERSION}.zip"
sudo mv tofu /usr/local/bin/
```

## Update Project Configuration

```hcl
# versions.tf - Update version constraints
terraform {
  # Allow 1.11 and above within the 1.x range
  required_version = ">= 1.11.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.60"
    }

    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.100"
    }
  }
}
```

```bash
# Update .opentofu-version file
echo "1.11.0" > .opentofu-version
```

## Running the Upgrade

```bash
# Reinitialize to update providers
tofu init -upgrade

# Check for deprecation warnings
tofu validate

# Fix any formatting issues
tofu fmt -recursive

# Run tests
tofu test -verbose

# Run a plan
tofu plan

# If everything looks good, apply (in non-prod first)
tofu apply
```

## Updating CI/CD Pipelines

```yaml
# .github/workflows/terraform.yml
name: Infrastructure Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu 1.11
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.11.0"

      - run: tofu init
      - run: tofu validate
      - run: tofu fmt -check -recursive

  plan:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup OpenTofu 1.11
        uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.11.0"

      - run: tofu init
      - run: tofu plan
```

## Post-Upgrade Validation

```bash
# Final validation steps
tofu version     # Confirm 1.11.0
tofu validate    # No errors
tofu plan        # No unexpected changes

# Commit the upgrade
git add versions.tf .opentofu-version .github/
git commit -m "Upgrade OpenTofu from 1.10 to 1.11

- Updated required_version to >= 1.11.0, < 2.0.0
- Updated .opentofu-version to 1.11.0
- Updated CI/CD workflows to use 1.11.0
- No configuration changes required"

# Open a PR for team review
git push origin feature/upgrade-opentofu-1.11
```

## Rollback

```bash
# Quick rollback
tofuenv use 1.10.x
echo "1.10.x" > .opentofu-version
git checkout -- versions.tf
tofu init
tofu validate
```

## Conclusion

Upgrading from OpenTofu 1.10 to 1.11 follows the same structured approach used for previous upgrades. Using a dedicated branch for the upgrade makes it easy to review changes, and the systematic testing approach ensures no regressions. With proper preparation, the upgrade process typically takes less than an hour for most infrastructure projects.
