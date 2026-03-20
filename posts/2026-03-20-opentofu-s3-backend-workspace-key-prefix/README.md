# How to Configure S3 Backend with workspace_key_prefix in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Backends, AWS, Workspaces

Description: Learn how to configure the workspace_key_prefix option in the OpenTofu S3 backend to organize workspace-specific state files in a custom directory structure.

## Introduction

When using workspaces with the S3 backend, OpenTofu stores workspace state files under a configurable prefix. The `workspace_key_prefix` option controls this prefix, allowing you to organize workspace state files in a meaningful directory structure within your S3 bucket.

## Default Workspace Key Structure

Without configuration, OpenTofu uses `env:` as the workspace key prefix:

```
s3://my-tofu-state/
├── production/terraform.tfstate         (default workspace)
└── env:/
    ├── staging/production/terraform.tfstate
    └── dev/production/terraform.tfstate
```

## Custom workspace_key_prefix

```hcl
terraform {
  backend "s3" {
    bucket               = "my-tofu-state"
    key                  = "terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "workspaces"
  }
}
```

This results in:

```
s3://my-tofu-state/
├── terraform.tfstate                          (default workspace)
└── workspaces/
    ├── staging/terraform.tfstate
    └── dev/terraform.tfstate
```

## Organizing by Application and Environment

```hcl
terraform {
  backend "s3" {
    bucket               = "acme-corp-terraform-state"
    key                  = "infrastructure/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "environments"
    encrypt              = true
    dynamodb_table       = "tofu-state-lock"
  }
}
```

Workspaces and state paths:

```
s3://acme-corp-terraform-state/
├── infrastructure/terraform.tfstate              (default = production)
└── environments/
    ├── staging/infrastructure/terraform.tfstate
    └── dev/infrastructure/terraform.tfstate
```

## Per-Application State Organization

Each application uses a different key but shares the workspace prefix convention:

```hcl
# apps/api/backend.tf
terraform {
  backend "s3" {
    bucket               = "acme-terraform-state"
    key                  = "apps/api/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "envs"
  }
}

# apps/frontend/backend.tf
terraform {
  backend "s3" {
    bucket               = "acme-terraform-state"
    key                  = "apps/frontend/terraform.tfstate"
    region               = "us-east-1"
    workspace_key_prefix = "envs"
  }
}
```

Results in:

```
s3://acme-terraform-state/
├── apps/
│   ├── api/terraform.tfstate
│   └── frontend/terraform.tfstate
└── envs/
    ├── staging/apps/api/terraform.tfstate
    ├── staging/apps/frontend/terraform.tfstate
    ├── dev/apps/api/terraform.tfstate
    └── dev/apps/frontend/terraform.tfstate
```

## Using Workspace Name in Configuration

Access the current workspace name in configuration:

```hcl
locals {
  is_production = terraform.workspace == "default"
  environment   = terraform.workspace == "default" ? "production" : terraform.workspace
}

resource "aws_instance" "app" {
  instance_type = local.is_production ? "m5.large" : "t3.small"

  tags = {
    Environment = local.environment
    Workspace   = terraform.workspace
  }
}
```

## Listing Workspace State Files

```bash
# See all workspace state files
aws s3 ls s3://my-tofu-state/workspaces/ --recursive

# Switch to a workspace and verify state
tofu workspace select staging
tofu state list
```

## Conclusion

`workspace_key_prefix` organizes workspace state files under a meaningful directory in your S3 bucket. Choose a descriptive prefix like `workspaces` or `environments` to make the S3 structure self-documenting. This is especially valuable when multiple applications share the same state bucket and you want clear visual separation between workspace and non-workspace state.
