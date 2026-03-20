# How to Understand When Not to Use Workspaces in OpenTofu - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Workspaces, Best Practices

Description: Learn the limitations and anti-patterns of OpenTofu workspaces to understand when separate configurations and state files are better choices than workspaces.

## Introduction

Workspaces are a useful tool, but they're not appropriate for every multi-environment scenario. Understanding their limitations helps you make better architectural decisions about when to use separate configurations instead.

## When Workspaces Work Well

Workspaces are well-suited when:
- Environments are structurally identical (same resource types, same topology)
- Only scale differs (instance sizes, counts)
- Environments are ephemeral (feature branches, test environments)
- The same team manages all environments

## Anti-Pattern 1: Different Resource Topologies

Workspaces share a single configuration. If your environments have fundamentally different architectures, workspaces force you into awkward conditional logic:

```hcl
# Anti-pattern: Too much conditional logic

resource "aws_eks_cluster" "main" {
  count = terraform.workspace == "production" ? 1 : 0
  # Only production has EKS
}

resource "aws_instance" "dev_app" {
  count = terraform.workspace == "development" ? 1 : 0
  # Dev uses a single EC2 instance
}

resource "aws_ecs_service" "staging_app" {
  count = terraform.workspace == "staging" ? 1 : 0
  # Staging uses ECS
}
```

**Better approach**: Separate configurations in separate directories:

```text
infrastructure/
├── development/     # EC2-based setup
│   ├── main.tf
│   └── backend.tf
├── staging/         # ECS-based setup
│   ├── main.tf
│   └── backend.tf
└── production/      # EKS-based setup
    ├── main.tf
    └── backend.tf
```

## Anti-Pattern 2: Different Teams, Different Access

Workspaces share the same configuration and same backend bucket (though different state files). If different teams own different environments, they shouldn't share a configuration:

```text
Problem: Platform team owns production, dev team owns development
Both in the same workspace = both can accidentally apply to wrong environment
```

**Better approach**: Separate repositories or directories with separate IAM policies:

```text
infrastructure/
├── platform-team/
│   └── production/     # Only platform team has IAM access to this backend
└── dev-team/
    └── development/    # Only dev team has IAM access to this backend
```

## Anti-Pattern 3: Sensitive Configuration Differences

If production requires additional security controls not present in lower environments, workspace conditionals become error-prone:

```hcl
# Anti-pattern: Security-critical conditional
resource "aws_s3_bucket_public_access_block" "app" {
  # What if this condition has a bug?
  block_public_acls = terraform.workspace == "production" ? true : false
}
```

**Better approach**: Enable strict controls in all environments and use environment-specific `.tfvars` for tuning.

## Anti-Pattern 4: Per-Customer or Per-Tenant Infrastructure

Each customer gets their own "environment" - this scales to hundreds of workspaces:

```bash
# Anti-pattern: Workspace per customer
tofu workspace new customer-acme
tofu workspace new customer-globex
tofu workspace new customer-initech
# ... 500 more workspaces
```

**Better approach**: Use `for_each` with a module to manage all tenants from one configuration, or separate state files per customer with a CI/CD orchestration layer.

## Anti-Pattern 5: Compliance Boundary Separation

Production and non-production must be in separate AWS accounts for compliance:

```hcl
# Cannot use workspaces when accounts differ
provider "aws" {
  # Workspaces can't easily switch the provider's account
  region = "us-east-1"
}
```

**Better approach**: Separate configurations with `provider "aws" { assume_role { role_arn = ... }}` per environment.

## The Rule of Thumb

Use workspaces when environments are **copies of each other at different scales**.

Use separate configurations when environments have:
- Different architectures
- Different team ownership
- Different compliance requirements
- Different AWS accounts/subscriptions
- Significantly different security postures

## Refactoring from Workspaces to Separate Configs

If you've outgrown workspaces:

```bash
# 1. Export workspace states
tofu workspace select production
tofu state pull > production.tfstate

tofu workspace select staging
tofu state pull > staging.tfstate

# 2. Create separate configuration directories
mkdir -p infrastructure/production
mkdir -p infrastructure/staging

# 3. Copy configuration with environment-specific settings
cp main.tf infrastructure/production/
cp production.tfvars infrastructure/production/terraform.tfvars

# 4. Configure separate backends
# 5. Push state to new backends
# 6. Validate with tofu plan (should show no changes)
```

## Conclusion

Workspaces are powerful for identical, scale-differentiated environments, but they're not a universal solution. When environments differ significantly in structure, ownership, or compliance requirements, separate configurations with dedicated backends provide better isolation, clearer ownership, and fewer opportunities for accidental cross-environment changes. Choose the right tool for your specific organizational and architectural context.
