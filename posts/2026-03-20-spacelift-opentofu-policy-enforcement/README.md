# How to Use Spacelift with OpenTofu for Policy Enforcement

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Spacelift, Policy Enforcement, OPA, Infrastructure as Code, DevOps

Description: Learn how to connect Spacelift to your OpenTofu stacks and use its Open Policy Agent-based policies to enforce security, cost, and governance rules before infrastructure changes reach production.

## Introduction

Spacelift is a CI/CD platform purpose-built for infrastructure-as-code. Its standout feature is policy enforcement: every `tofu plan` output can be evaluated against Open Policy Agent (OPA) policies that can warn, block, or auto-approve changes based on your organization's rules.

## Creating a Spacelift Stack for OpenTofu

In the Spacelift UI (or via its Terraform/OpenTofu provider), create a stack that uses the OpenTofu runner:

```hcl
# spacelift.tf — manage Spacelift stacks with OpenTofu
terraform {
  required_providers {
    spacelift = {
      source  = "spacelift-io/spacelift"
      version = "~> 1.0"
    }
  }
}

provider "spacelift" {}

resource "spacelift_stack" "production_infra" {
  name       = "production-infra"
  repository = "my-org/infra-repo"
  branch     = "main"
  project_root = "environments/production"

  # Use OpenTofu instead of Terraform
  opentofu_version = "1.9.0"

  # Auto-apply on push to main (use with caution in production)
  autodeploy = false
}
```

## Writing an OPA Policy in Spacelift

Spacelift policies are written in Rego. This example denies any plan that destroys more than 5 resources:

```rego
# policies/deny-mass-destroy.rego
package spacelift

# Collect all resources that will be destroyed
destroyed_resources := [resource |
    resource := input.terraform.resource_changes[_]
    resource.change.actions[_] == "delete"
]

# Deny if more than 5 resources would be destroyed at once
deny["Mass destruction: more than 5 resources would be deleted"] {
    count(destroyed_resources) > 5
}

# Warn if any resource change involves a database instance
warn["Database instance change detected — review carefully"] {
    resource := input.terraform.resource_changes[_]
    contains(resource.type, "db_instance")
    resource.change.actions[_] != "no-op"
}
```

## Enforcing Mandatory Tags Policy

```rego
# policies/require-tags.rego
package spacelift

required_tags := {"Environment", "Owner", "CostCenter"}

# Collect resources missing required tags
resources_missing_tags := [resource.address |
    resource := input.terraform.resource_changes[_]
    resource.change.actions[_] != "delete"
    resource.change.actions[_] != "no-op"
    remaining := required_tags - {tag | tag := resource.change.after.tags[_]}
    count(remaining) > 0
]

deny[msg] {
    count(resources_missing_tags) > 0
    msg := sprintf("Resources missing required tags: %v", [resources_missing_tags])
}
```

## Attaching a Policy to a Stack

```hcl
# Create the policy resource
resource "spacelift_policy" "deny_mass_destroy" {
  name = "deny-mass-destroy"
  body = file("policies/deny-mass-destroy.rego")
  type = "PLAN"
}

# Attach it to the stack
resource "spacelift_policy_attachment" "production_deny_mass_destroy" {
  policy_id = spacelift_policy.deny_mass_destroy.id
  stack_id  = spacelift_stack.production_infra.id
}
```

## Environment Variables and Secrets

```hcl
# Inject AWS credentials as environment variables into the stack
resource "spacelift_environment_variable" "aws_region" {
  stack_id   = spacelift_stack.production_infra.id
  name       = "AWS_DEFAULT_REGION"
  value      = "us-east-1"
  write_only = false
}

resource "spacelift_environment_variable" "aws_role" {
  stack_id   = spacelift_stack.production_infra.id
  name       = "AWS_ROLE_ARN"
  value      = "arn:aws:iam::123456789012:role/spacelift-role"
  write_only = false
}
```

## Approval Policies

Use an approval policy to require human sign-off before applying:

```rego
# policies/require-approval.rego
package spacelift

# Require at least one approval for production stacks
approve { input.reviews.approvals >= 1 }
```

## Conclusion

Spacelift provides a managed GitOps platform for OpenTofu with built-in OPA policy enforcement. By attaching plan, access, and approval policies to your stacks, you get automated guardrails that prevent destructive changes, enforce tagging standards, and require human review — all before a single resource is modified in production.
