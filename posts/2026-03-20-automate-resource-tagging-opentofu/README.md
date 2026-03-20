# How to Automate Resource Tagging with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, AWS, Tagging, Cost Management, Governance, Infrastructure as Code

Description: Learn how to implement consistent resource tagging across all OpenTofu-managed resources using default tags, local variables, and provider-level tag defaults.

## Introduction

Consistent resource tagging enables cost allocation, security policy enforcement, and automated governance. OpenTofu provides provider-level default tags, local tag variables, and merge patterns to ensure every resource gets the right tags without repetition.

## Provider-Level Default Tags

AWS provider supports `default_tags` that apply to all resources automatically.

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment  = var.environment
      Project      = var.project
      Owner        = var.team
      ManagedBy    = "opentofu"
      CostCenter   = var.cost_center
      Terraform    = "true"
    }
  }
}
```

## Centralized Tag Local Variables

```hcl
# modules/tagging/main.tf
variable "environment" { type = string }
variable "project"     { type = string }
variable "team"        { type = string }
variable "cost_center" { type = string }
variable "extra_tags"  {
  type    = map(string)
  default = {}
}

locals {
  # Base tags applied to all resources
  base_tags = {
    Environment = var.environment
    Project     = var.project
    Team        = var.team
    CostCenter  = var.cost_center
    ManagedBy   = "opentofu"
    CreatedAt   = timestamp()
  }

  # Merge base tags with resource-specific extra tags
  # Extra tags override base tags on conflict
  all_tags = merge(local.base_tags, var.extra_tags)
}

output "tags" {
  value = local.all_tags
}
```

## Using the Tagging Module

```hcl
module "tags" {
  source      = "./modules/tagging"
  environment = var.environment
  project     = "payment-service"
  team        = "platform"
  cost_center = "engineering-001"

  extra_tags = {
    Component = "database"
    BackupEnabled = "true"
  }
}

resource "aws_db_instance" "main" {
  identifier = "payments-db"
  # ... other config ...

  tags = module.tags.tags
}

resource "aws_s3_bucket" "backups" {
  bucket = "payments-backups-${var.environment}"

  tags = merge(module.tags.tags, {
    DataClassification = "confidential"
    RetentionDays      = "90"
  })
}
```

## Enforcing Required Tags with check Blocks

OpenTofu 1.5+ supports `check` blocks for post-apply validation.

```hcl
check "all_resources_tagged" {
  assert {
    condition = length([
      for resource in [
        aws_instance.web.tags,
        aws_db_instance.main.tags,
        aws_s3_bucket.data.tags
      ] : resource
      if !contains(keys(resource), "CostCenter")
    ]) == 0
    error_message = "All resources must have a CostCenter tag."
  }
}
```

## Automated Tag Compliance Script

```bash
#!/bin/bash
# scripts/check-tag-compliance.sh
# Scan OpenTofu state for untagged resources

REQUIRED_TAGS=("Environment" "Project" "Team" "CostCenter")
VIOLATIONS=0

# Get all resources from state
resources=$(tofu state list)

for resource in $resources; do
  # Get tags for this resource
  tags=$(tofu state show "$resource" 2>/dev/null | grep -A 50 "tags " | grep -v "^tags_all" || true)

  for tag in "${REQUIRED_TAGS[@]}"; do
    if ! echo "$tags" | grep -q "\"$tag\""; then
      echo "MISSING TAG '$tag' on: $resource"
      VIOLATIONS=$((VIOLATIONS + 1))
    fi
  done
done

if [[ $VIOLATIONS -gt 0 ]]; then
  echo "Found $VIOLATIONS tag violations."
  exit 1
fi

echo "All resources comply with tagging policy."
```

## Variable File with Tags

```hcl
# environments/prod.tfvars
environment = "prod"
project     = "payment-service"
team        = "platform-eng"
cost_center = "cc-1234"
```

## Summary

Consistent resource tagging requires a combination of provider-level default tags, centralized tag modules, and compliance checks. OpenTofu's `default_tags` in the AWS provider eliminates tag repetition, while `check` blocks and compliance scripts enforce tag governance across the entire resource fleet.
