# How to Use required_version to Enforce OpenTofu Versions - A Practical Guide

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Version Constraints, Required_version, Infrastructure as Code, DevOps

Description: A guide to using the required_version constraint in OpenTofu to prevent configuration from being applied with incompatible tool versions.

## Introduction

The `required_version` setting in the `terraform` block enforces that configurations are only applied with compatible OpenTofu versions. This acts as a safety net preventing someone from accidentally running your infrastructure code with an incompatible version that might behave differently.

## Basic Syntax

```hcl
# versions.tf

terraform {
  required_version = ">= 1.9.0"
}
```

## Version Constraint Operators

```hcl
terraform {
  # Allow any version >= 1.9.0
  required_version = ">= 1.9.0"

  # Allow exactly 1.9.0 only
  required_version = "= 1.9.0"

  # Allow any 1.9.x version (patch-level changes only)
  required_version = "~> 1.9.0"

  # Allow any 1.x.x version (minor and patch changes)
  required_version = "~> 1.9"

  # Allow within a range
  required_version = ">= 1.9.0, < 2.0.0"

  # Allow multiple specific versions
  required_version = ">= 1.8.0, < 1.10.0"
}
```

## Constraint Operator Reference

| Operator | Meaning | Example |
|----------|---------|---------|
| `=`  | Exact version | `= 1.9.0` |
| `!=` | Exclude version | `!= 1.9.1` |
| `>`  | Greater than | `> 1.9.0` |
| `>=` | Greater than or equal | `>= 1.9.0` |
| `<`  | Less than | `< 2.0.0` |
| `<=` | Less than or equal | `<= 1.9.5` |
| `~>` | Pessimistic (allows patch/minor) | `~> 1.9.0` |

## The Pessimistic Constraint Operator

The `~>` operator is commonly used because it allows bug fixes while preventing breaking changes:

```hcl
terraform {
  # ~> 1.9.0 allows: 1.9.0, 1.9.1, 1.9.2, ...
  # But NOT: 1.10.0 or 2.0.0
  required_version = "~> 1.9.0"
}
```

```hcl
terraform {
  # ~> 1.9 allows: 1.9.0, 1.9.1, 1.10.0, 1.11.0, ...
  # But NOT: 2.0.0
  required_version = "~> 1.9"
}
```

## Real-World Example: Production Configuration

```hcl
# versions.tf
terraform {
  # Allow 1.9.x and 1.10.x but not 2.x
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }

    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.27"
    }

    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}
```

## What Happens When Version Doesn't Match

```bash
# If you run with an incompatible version, OpenTofu stops immediately:
tofu init
# Error: OpenTofu v1.8.5 does not satisfy the required version constraint ">= 1.9.0"
# OpenTofu is at version 1.8.5, but the configuration requires at least version 1.9.0.

tofu plan
# Error: OpenTofu v1.8.5 does not satisfy the required version constraint ">= 1.9.0"
```

## Version Constraints in Modules

```hcl
# modules/networking/versions.tf
terraform {
  # Module version requirements can be more permissive
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 4.0, < 6.0"
    }
  }
}
```

## Best Practices

```hcl
# versions.tf - Best practices for team projects
terraform {
  # Use a range that:
  # 1. Requires minimum features you depend on
  # 2. Doesn't pin to an exact patch version (allows bug fixes)
  # 3. Excludes future major versions (potential breaking changes)
  required_version = ">= 1.9.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      # Pin minor version for stability
      version = "~> 5.30"
    }
  }
}
```

## Checking Constraint Satisfaction

```bash
# Check if your current version satisfies the constraint
tofu version
# OpenTofu v1.9.0

# Try running with mismatched version (using tofuenv)
tofuenv use 1.8.5
tofu validate
# Error: OpenTofu v1.8.5 does not satisfy the required version constraint
```

## Conclusion

The `required_version` constraint is a simple but powerful way to prevent infrastructure misconfigurations caused by version mismatches. By combining explicit version constraints in HCL with `.opentofu-version` files for tooling, you create a robust version enforcement system that protects your infrastructure from accidental changes caused by incompatible tool versions.
