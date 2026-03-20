# How to Find and Evaluate OpenTofu Registry Modules

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Registry, Modules, Best Practices, Community, Infrastructure as Code

Description: Learn how to search, evaluate, and safely adopt modules from the OpenTofu Registry in your infrastructure configurations.

## Introduction

The OpenTofu Registry hosts thousands of community and verified modules for common infrastructure patterns. Knowing how to evaluate module quality before using it in production saves time and prevents issues.

## Finding Modules

```bash
# Search the OpenTofu Registry
# https://registry.opentofu.org/modules

# Filter by provider, verified status, or download count

# Or use the CLI to explore module versions
# (modules are referenced in your configuration)
```

## Using a Registry Module

```hcl
# Reference a module from the registry
module "vpc" {
  source  = "registry.opentofu.org/terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # always pin with version constraints

  name = "${var.app_name}-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"

  tags = {
    Environment = var.environment
    ManagedBy   = "opentofu"
  }
}
```

## Evaluation Checklist

Before using a module in production, evaluate it against these criteria:

### 1. Maintenance Activity
```bash
# Check last commit date on GitHub
# Look for active issue responses
# Check if the module has recent releases
```

### 2. Download Count and Stars
High download counts and stars indicate community trust.
- 100K+ downloads: widely used
- 1K-100K: moderate adoption
- < 1K: use with caution

### 3. Documentation Quality
- Does the README explain all inputs and outputs?
- Are there usage examples for common scenarios?
- Is there a CHANGELOG?

### 4. Test Coverage
```bash
# Check if the module has tests
ls test/
# or
ls examples/
```

### 5. Code Quality
```hcl
# Review the source on GitHub before using
# Look for:
# - Consistent naming conventions
# - Input validation with variable validation blocks
# - Sensible defaults
# - No hardcoded values
# - Proper use of outputs

# Example of a well-written module input:
variable "instance_type" {
  type        = string
  description = "EC2 instance type for the web servers"
  default     = "t3.medium"

  validation {
    condition     = can(regex("^(t3|m5|c5)\\.", var.instance_type))
    error_message = "instance_type must be a t3, m5, or c5 instance type."
  }
}
```

### 6. Security Considerations
```hcl
# Check for:
# - No hardcoded credentials
# - Encryption enabled by default
# - Least-privilege IAM policies
# - Security group rules are restrictive
# - Public access is disabled by default
```

## Pinning Module Versions

Always pin module versions to avoid unexpected changes.

```hcl
# Bad: unpinned (gets latest on every init)
module "vpc" {
  source = "registry.opentofu.org/terraform-aws-modules/vpc/aws"
}

# Good: pinned to minor version
module "vpc" {
  source  = "registry.opentofu.org/terraform-aws-modules/vpc/aws"
  version = "~> 5.1"
}

# Best for production: pin exactly
module "vpc" {
  source  = "registry.opentofu.org/terraform-aws-modules/vpc/aws"
  version = "5.1.2"
}
```

## Vendoring Modules

For environments without internet access, vendor modules locally.

```bash
# After init, modules are cached in .terraform/modules/
# For air-gapped environments, copy this directory with your code

# Or use a private registry / Git source
module "vpc" {
  source = "git::https://github.com/your-org/terraform-aws-vpc.git?ref=v5.1.2"
}
```

## Summary

Registry modules accelerate infrastructure development, but quality varies significantly. Evaluate modules by checking maintenance activity, download counts, documentation quality, test coverage, and security defaults. Always pin module versions, review the source before production use, and prefer modules from verified publishers or well-known community organizations.
