# How to Understand Variable Precedence in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Variables, Precedence, Infrastructure as Code, DevOps

Description: A comprehensive guide to understanding how OpenTofu resolves variable values when multiple sources provide the same variable.

## Introduction

When multiple sources provide values for the same variable, OpenTofu uses a defined precedence order to determine which value to use. Understanding this precedence prevents unexpected behavior and helps you design reliable variable management strategies.

## Precedence Order (Lowest to Highest)

```
1. Default values (in variable blocks)       <- LOWEST
2. terraform.tfvars
3. terraform.tfvars.json
4. *.auto.tfvars (alphabetical order)
5. *.auto.tfvars.json (alphabetical order)
6. -var-file flags (in order specified)
7. -var flags (in order specified)
8. TF_VAR_ environment variables             <- HIGHEST (for CLI)
```

## Demonstration

```hcl
# variables.tf
variable "instance_count" {
  type    = number
  default = 1  # Level 1: Default
}
```

```hcl
# terraform.tfvars
instance_count = 2  # Level 2: Overrides default
```

```hcl
# common.auto.tfvars
instance_count = 3  # Level 4: Overrides terraform.tfvars
```

```bash
# The following overrides all file-based values:
tofu plan -var="instance_count=5"  # Level 7: Overrides auto.tfvars

# Check which value is used:
tofu console
> var.instance_count
# Output: 5
```

## Step-by-Step Precedence Example

```bash
# Setup:
# - default in variable block: 1
# - terraform.tfvars: 2
# - common.auto.tfvars: 3
# - -var-file="override.tfvars" (value=4): 4
# - -var="instance_count=5": 5
# - TF_VAR_instance_count=6: 6

# Without any explicit variables (uses default=1):
unset TF_VAR_instance_count
tofu plan  # instance_count = 2 (terraform.tfvars takes over from default)

# With TF_VAR (overrides everything):
export TF_VAR_instance_count=6
tofu plan  # instance_count = 6

# With both -var and TF_VAR (-var wins):
tofu plan -var="instance_count=5"  # instance_count = 5 (-var > TF_VAR)
```

## Multiple -var-file Loading Order

```bash
# When using multiple -var-file flags, later files override earlier ones
tofu apply \
  -var-file="base.tfvars" \       # 1st: base values
  -var-file="environment.tfvars"  # 2nd: overrides base
  -var="specific_override=value"  # 3rd: overrides both files
```

## Auto.tfvars Alphabetical Loading

```hcl
# a-base.auto.tfvars
instance_type = "t3.micro"

# b-compute.auto.tfvars (loaded after a-base)
instance_type = "t3.small"  # Overrides a-base.auto.tfvars

# Result: instance_type = "t3.small"
```

## Practical Examples

```bash
# Development workflow - mostly defaults and terraform.tfvars
cd dev/
tofu plan  # Uses defaults + terraform.tfvars

# Staging override from CI/CD
tofu plan \
  -var-file="staging.tfvars" \
  -var="image_version=$BUILD_TAG"

# Production with secrets from environment
export TF_VAR_db_password="$(vault read -field=value secret/prod/db)"
tofu apply -var-file="prod.tfvars" -auto-approve
```

## Debugging Variable Values

```bash
# Check the effective value of variables
tofu console
> var.instance_count
> var.environment

# Or print in a plan
tofu plan
# Shows the computed values in the plan output
```

## Common Gotchas

```bash
# Gotcha 1: TF_VAR_ set from a previous session
export TF_VAR_environment="prod"
# Forgot it's set, now running in dev context!
tofu plan  # Uses "prod" because TF_VAR_ overrides terraform.tfvars

# Fix: Always unset TF_VAR_ when switching contexts
unset TF_VAR_environment

# Gotcha 2: -var overrides -var-file order doesn't matter
# -var always takes precedence over -var-file regardless of order
tofu plan -var="count=5" -var-file="values.tfvars"
# count=5 even if values.tfvars has count=10
```

## Conclusion

Understanding variable precedence is crucial for predictable OpenTofu behavior. The key insight is that more explicit sources (like `-var` flags) override less explicit ones (like file-based defaults), creating a clear hierarchy from most general (defaults) to most specific (command-line flags). Design your variable strategy by placing commonly shared values in `terraform.tfvars` and using `-var` flags or `TF_VAR_` variables only for environment-specific or sensitive overrides.
