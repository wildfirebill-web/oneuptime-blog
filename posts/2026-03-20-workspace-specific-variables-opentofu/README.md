# How to Pass Workspace-Specific Variables in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Workspaces, Variable, tfvars, Infrastructure as Code

Description: Learn multiple patterns for passing different variable values to each OpenTofu workspace without duplicating configuration logic.

When you manage multiple environments with workspaces, each environment typically needs different values - different instance sizes, replica counts, domain names, or feature flags. This guide covers the main strategies for passing workspace-specific variables cleanly.

## Strategy 1: Per-Workspace tfvars Files

Maintain one `.tfvars` file per workspace and select the right one at apply time:

```text
infra/
├── main.tf
├── variables.tf
├── vars/
│   ├── default.tfvars
│   ├── staging.tfvars
│   └── production.tfvars
```

```hcl
# vars/staging.tfvars

instance_type    = "t3.medium"
min_capacity     = 2
max_capacity     = 5
db_instance_class = "db.t3.medium"
enable_deletion_protection = false
```

```hcl
# vars/production.tfvars
instance_type    = "m5.large"
min_capacity     = 5
max_capacity     = 50
db_instance_class = "db.r5.large"
enable_deletion_protection = true
```

Apply using the file that matches the current workspace:

```bash
WORKSPACE=$(tofu workspace show)
tofu apply -var-file="vars/${WORKSPACE}.tfvars"
```

## Strategy 2: locals Map Keyed by Workspace

Keep all values in a single file using a map and `terraform.workspace` as the lookup key:

```hcl
# locals.tf
locals {
  env_vars = {
    staging = {
      instance_type             = "t3.medium"
      min_capacity              = 2
      max_capacity              = 5
      db_instance_class         = "db.t3.medium"
      enable_deletion_protection = false
    }
    production = {
      instance_type             = "m5.large"
      min_capacity              = 5
      max_capacity              = 50
      db_instance_class         = "db.r5.large"
      enable_deletion_protection = true
    }
  }

  # Fall back to staging defaults for unrecognized workspaces (e.g. dev-*)
  vars = lookup(local.env_vars, terraform.workspace, local.env_vars["staging"])
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = local.vars.instance_type
}
```

## Strategy 3: TF_VAR_ Environment Variables in CI

Set environment variables in your CI pipeline to inject workspace-specific values:

```bash
# In a CI step for the staging workspace
export TF_VAR_instance_type="t3.medium"
export TF_VAR_min_capacity=2
export TF_VAR_db_instance_class="db.t3.medium"

tofu workspace select staging
tofu apply -auto-approve
```

This is especially useful when values come from secrets managers or pipeline parameters.

## Strategy 4: Variable Defaults with Workspace Overrides

Define sensible defaults in `variables.tf` and allow workspace-level overrides:

```hcl
# variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"  # Safe default for all workspaces
}

variable "enable_deletion_protection" {
  description = "Enable deletion protection on the RDS instance"
  type        = bool
  default     = false
}
```

Production overrides are provided via a `production.tfvars` file applied only in the production workspace.

## Combining Strategies in a CI Script

A robust CI script selects the right strategy automatically:

```bash
#!/usr/bin/env bash
# ci-apply.sh
set -euo pipefail

WORKSPACE="${1:-$(tofu workspace show)}"
VARS_FILE="vars/${WORKSPACE}.tfvars"

tofu workspace select "$WORKSPACE" || tofu workspace new "$WORKSPACE"

if [[ -f "$VARS_FILE" ]]; then
  echo "Applying with $VARS_FILE"
  tofu apply -var-file="$VARS_FILE" -auto-approve
else
  echo "No vars file for $WORKSPACE, using defaults"
  tofu apply -auto-approve
fi
```

## Validating That Required Variables Are Set

Add validation to `variables.tf` to catch missing values early:

```hcl
variable "db_instance_class" {
  description = "RDS instance class"
  type        = string

  validation {
    condition     = can(regex("^db\\.", var.db_instance_class))
    error_message = "db_instance_class must start with 'db.' (e.g., db.t3.medium)"
  }
}
```

## Conclusion

Workspace-specific variables are best handled through a combination of per-workspace `.tfvars` files for clear auditability and a `locals` map for shared, in-code defaults. Choose the strategy that fits your team's workflow, and automate the variable file selection in CI to keep deployments consistent.
