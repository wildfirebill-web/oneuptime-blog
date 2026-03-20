# How to Configure S3 Backend with workspace_key_prefix in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, AWS, State Management, Workspaces

Description: Learn how to use the workspace_key_prefix setting in the OpenTofu S3 backend to customize how workspace state files are organized within your S3 bucket.

## Introduction

When using OpenTofu workspaces with the S3 backend, workspace state files are stored in a subdirectory structure. The `workspace_key_prefix` parameter controls the prefix used for this directory — by default it's `env:`, but you can customize it to match your organization's naming conventions.

## Default Workspace Key Structure

Without `workspace_key_prefix`, workspaces use `env:` as the prefix:

```
S3 bucket structure (default):
my-terraform-state/
├── prod/terraform.tfstate           ← default workspace
└── env:/
    ├── staging/
    │   └── prod/terraform.tfstate   ← staging workspace
    └── development/
        └── prod/terraform.tfstate   ← development workspace
```

## Customizing workspace_key_prefix

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket   = "my-terraform-state"
    key      = "app/terraform.tfstate"  # Base key path
    region   = "us-east-1"
    encrypt  = true

    # Custom workspace prefix
    workspace_key_prefix = "workspaces"
  }
}
```

With this configuration:

```
S3 bucket structure:
my-terraform-state/
├── app/terraform.tfstate                    ← default workspace
└── workspaces/
    ├── production/
    │   └── app/terraform.tfstate
    ├── staging/
    │   └── app/terraform.tfstate
    └── development/
        └── app/terraform.tfstate
```

## Examples of Custom Prefixes

### Environment-Based Prefix

```hcl
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "app/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "environments"
  }
}

# Results in: environments/production/app/terraform.tfstate
```

### Team-Based Prefix

```hcl
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "networking/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "teams/platform"
  }
}

# Results in: teams/platform/staging/networking/terraform.tfstate
```

### No Prefix (Flat Structure)

```hcl
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = ""  # Empty = no prefix directory
  }
}

# Results in: production/terraform.tfstate
#             staging/terraform.tfstate
```

## Using workspace_key_prefix with Multiple Configurations

Organize multiple configurations within the same bucket:

```hcl
# networking/backend.tf
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "networking/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "envs"
    dynamodb_table        = "terraform-state-locks"
  }
}

# compute/backend.tf
terraform {
  backend "s3" {
    bucket               = "my-terraform-state"
    key                  = "compute/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "envs"
    dynamodb_table        = "terraform-state-locks"
  }
}
```

```
S3 structure:
├── networking/terraform.tfstate          ← networking/default
├── compute/terraform.tfstate             ← compute/default
└── envs/
    ├── production/
    │   ├── networking/terraform.tfstate
    │   └── compute/terraform.tfstate
    └── staging/
        ├── networking/terraform.tfstate
        └── compute/terraform.tfstate
```

## Listing Workspace State Files

```bash
# List all workspace state files
aws s3 ls s3://my-terraform-state/workspaces/ --recursive | grep terraform.tfstate

# With a specific prefix
aws s3 ls s3://my-terraform-state/envs/ --recursive
```

## Switching Workspaces

```bash
# Create and switch to workspace
tofu workspace new production

# The state file is automatically created at the correct path
# e.g., workspaces/production/app/terraform.tfstate

tofu workspace list
tofu workspace select staging
```

## Migrating workspace_key_prefix

If you change `workspace_key_prefix`, you must move existing state files:

```bash
# Before: using "env:" prefix (default)
# After: using "environments" prefix

# Move existing workspace states
aws s3 mv \
  s3://my-bucket/env:/production/app/terraform.tfstate \
  s3://my-bucket/environments/production/app/terraform.tfstate

# Then update backend.tf and run tofu init -reconfigure
```

## Conclusion

The `workspace_key_prefix` parameter gives you control over how workspace state files are organized in your S3 bucket. Choose a meaningful prefix that aligns with your team's conventions — `environments`, `workspaces`, `envs`, or an empty string for a flat structure. Establish this convention early and keep it consistent across all configurations using the same bucket to maintain a clean, navigable state file hierarchy.
