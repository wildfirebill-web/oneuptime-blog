# How to Enforce Tagging Policies with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Tagging Policies, Governance, Compliance, Infrastructure as Code, FinOps

Description: Learn how to enforce mandatory resource tagging in OpenTofu through variable validation, OPA policies, and provider default_tags - ensuring every resource is properly labeled for cost allocation...

## Introduction

Mandatory resource tagging is one of the most impactful governance controls in cloud environments. It enables accurate cost allocation, security auditing, and compliance reporting. OpenTofu provides multiple layers to enforce tagging - from provider defaults to policy-as-code gates.

## Layer 1: Provider default_tags (AWS)

The most reliable enforcement - tags applied automatically at the provider level:

```hcl
variable "environment" { type = string }
variable "team"        { type = string }
variable "cost_center" { type = string }

provider "aws" {
  region = var.aws_region

  # These tags are applied to ALL resources automatically
  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      CostCenter  = var.cost_center
      ManagedBy   = "opentofu"
      Repository  = "my-org/infra-repo"
    }
  }
}
```

## Layer 2: Variable Validation for Tag Values

```hcl
variable "environment" {
  type = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "cost_center" {
  type = string
  validation {
    condition     = can(regex("^CC-[0-9]{4}$", var.cost_center))
    error_message = "CostCenter must follow the format CC-XXXX (e.g., CC-1234)."
  }
}
```

## Layer 3: OPA Policy to Block Missing Tags

```rego
# policies/require_tags.rego

package main

required_tags := {"Environment", "Team", "CostCenter", "ManagedBy"}

taggable_resources := {
    "aws_instance", "aws_db_instance", "aws_s3_bucket",
    "aws_eks_cluster", "aws_vpc", "aws_lambda_function"
}

deny[msg] {
    resource := input.resource_changes[_]
    resource.type in taggable_resources
    resource.change.actions[_] in {"create", "update"}

    present := {tag | resource.change.after.tags[tag]}
    missing  := required_tags - present
    count(missing) > 0

    msg := sprintf("Resource '%s' missing required tags: %v", [resource.address, missing])
}
```

```bash
# Run policy check before apply
tofu show -json tfplan.binary > tfplan.json
conftest test tfplan.json --policy policies/
```

## Layer 4: AWS Tag Policies (Organization Level)

```hcl
# Enforce tag policies at the AWS Organization level
resource "aws_organizations_policy" "tagging" {
  name        = "RequiredTags"
  type        = "TAG_POLICY"
  description = "Enforce required tags on all resources"

  content = jsonencode({
    tags = {
      Environment = {
        tag_key = { "@@assign" = "Environment" }
        tag_value = {
          "@@assign" = ["dev", "staging", "prod"]
        }
        enforced_for = {
          "@@assign" = ["ec2:instance", "rds:db", "s3:bucket"]
        }
      }
    }
  })
}
```

## Automated Tag Remediation

For resources that slip through without tags, automate detection and notification:

```bash
#!/bin/bash
# find-untagged.sh - find resources missing required tags
aws resourcegroupstaggingapi get-resources \
  --tag-filters "Key=ManagedBy" \
  --resource-type-filters ec2:instance \
  --query "ResourceTagMappingList[?Tags[?Key=='Environment']==null].ResourceARN" \
  --output text
```

## Conclusion

Enforcing tagging policies requires multiple complementary layers: provider `default_tags` ensures tags are applied even when developers forget, variable validation ensures valid tag values, OPA policies block deployments with missing tags, and AWS Tag Policies enforce compliance at the organization level. Together they make it nearly impossible to deploy untagged resources.
