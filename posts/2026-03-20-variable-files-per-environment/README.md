# How to Use Variable Files Per Environment in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variable, Environment, tfvars, Infrastructure as Code, DevOps

Description: A guide to managing environment-specific configurations using separate variable files in OpenTofu.

## Introduction

Managing multiple environments (dev, staging, production) is a common challenge in infrastructure management. Using separate variable files per environment is one of the most practical approaches for maintaining environment-specific configurations while sharing the same infrastructure code.

## Directory Structure

```text
infrastructure/
├── main.tf             # Shared infrastructure code
├── variables.tf        # Variable declarations
├── outputs.tf          # Outputs
├── versions.tf         # Versions and providers
├── locals.tf           # Computed local values
├── environments/
│   ├── dev.tfvars      # Development values
│   ├── staging.tfvars  # Staging values
│   └── prod.tfvars     # Production values
└── backends/
    ├── dev.hcl         # Dev backend config
    ├── staging.hcl     # Staging backend config
    └── prod.hcl        # Prod backend config
```

## Environment Variable Files

```hcl
# environments/dev.tfvars

environment    = "dev"
aws_region     = "us-east-1"
instance_type  = "t3.micro"
instance_count = 1
multi_az       = false

db_instance_class          = "db.t3.micro"
db_allocated_storage       = 20
db_backup_retention_period = 1

enable_deletion_protection = false
enable_enhanced_monitoring = false
```

```hcl
# environments/staging.tfvars
environment    = "staging"
aws_region     = "us-east-1"
instance_type  = "t3.small"
instance_count = 2
multi_az       = true

db_instance_class          = "db.t3.small"
db_allocated_storage       = 50
db_backup_retention_period = 7

enable_deletion_protection = false
enable_enhanced_monitoring = true
```

```hcl
# environments/prod.tfvars
environment    = "prod"
aws_region     = "us-east-1"
instance_type  = "t3.large"
instance_count = 4
multi_az       = true

db_instance_class          = "db.r5.large"
db_allocated_storage       = 500
db_backup_retention_period = 30

enable_deletion_protection = true
enable_enhanced_monitoring = true
```

## Backend Configuration Per Environment

```hcl
# backends/dev.hcl
bucket         = "myapp-tofu-state"
key            = "dev/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "tofu-state-lock"
```

```hcl
# backends/prod.hcl
bucket         = "myapp-tofu-state"
key            = "prod/terraform.tfstate"
region         = "us-east-1"
encrypt        = true
dynamodb_table = "tofu-state-lock"
```

## Deployment Scripts

```bash
#!/bin/bash
# deploy.sh - Deploy to a specific environment

ENVIRONMENT="${1:?Usage: $0 <dev|staging|prod>}"
ACTION="${2:-plan}"

# Validate environment
if [[ ! "$ENVIRONMENT" =~ ^(dev|staging|prod)$ ]]; then
  echo "Error: Environment must be dev, staging, or prod"
  exit 1
fi

echo "=== Deploying to: $ENVIRONMENT ==="

# Initialize with environment-specific backend
tofu init -reconfigure \
  -backend-config="backends/${ENVIRONMENT}.hcl"

# Plan or apply
if [ "$ACTION" = "apply" ]; then
  tofu apply \
    -var-file="environments/${ENVIRONMENT}.tfvars" \
    -auto-approve
else
  tofu plan \
    -var-file="environments/${ENVIRONMENT}.tfvars"
fi
```

```bash
# Usage:
./deploy.sh dev plan
./deploy.sh staging apply
./deploy.sh prod plan
```

## Makefile for Environment Deployments

```makefile
# Makefile

ENV ?= dev

.PHONY: init plan apply destroy

init:
	tofu init -reconfigure -backend-config=backends/$(ENV).hcl

plan: init
	tofu plan -var-file=environments/$(ENV).tfvars

apply: init
	tofu apply -var-file=environments/$(ENV).tfvars

destroy: init
	tofu destroy -var-file=environments/$(ENV).tfvars

fmt-check:
	tofu fmt -check -recursive

# Usage:
# make plan ENV=dev
# make apply ENV=staging
# make destroy ENV=dev
```

## CI/CD Pipeline Per Environment

```yaml
# .github/workflows/deploy.yml
name: Deploy Infrastructure

on:
  push:
    branches:
      - main        # -> prod
      - staging     # -> staging
      - develop     # -> dev

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Determine environment
        id: env
        run: |
          case "${{ github.ref_name }}" in
            main)    echo "ENVIRONMENT=prod" >> $GITHUB_OUTPUT ;;
            staging) echo "ENVIRONMENT=staging" >> $GITHUB_OUTPUT ;;
            develop) echo "ENVIRONMENT=dev" >> $GITHUB_OUTPUT ;;
          esac

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.9.0"

      - name: Deploy
        run: |
          tofu init -backend-config=backends/${{ steps.env.outputs.ENVIRONMENT }}.hcl
          tofu apply -var-file=environments/${{ steps.env.outputs.ENVIRONMENT }}.tfvars -auto-approve
```

## Conclusion

Using environment-specific variable files creates a clean, maintainable pattern for multi-environment infrastructure management. All environments share the same HCL code, while variable files provide the customization. Combining this with environment-specific backend configurations ensures state isolation between environments, preventing accidental cross-environment operations. This pattern scales from simple two-environment setups to complex multi-region, multi-account configurations.
