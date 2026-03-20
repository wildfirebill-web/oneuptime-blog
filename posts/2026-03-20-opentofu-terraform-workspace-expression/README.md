# How to Use the terraform.workspace Expression in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Workspaces

Description: Learn how to use the terraform.workspace expression in OpenTofu to make configuration dynamic based on the active workspace.

## Introduction

The `terraform.workspace` expression returns the name of the currently active workspace as a string. You can use it anywhere a string is valid in OpenTofu configuration — in resource arguments, locals, conditions, and tags. It is the primary way to make a single configuration behave differently per environment.

## Basic Usage

```hcl
# Returns "default", "staging", "production", etc.
output "current_workspace" {
  value = terraform.workspace
}
```

## Resource Naming

```hcl
# Include workspace in all resource names
resource "aws_s3_bucket" "data" {
  bucket = "acme-data-${terraform.workspace}"
}

resource "aws_dynamodb_table" "sessions" {
  name = "sessions-${terraform.workspace}"
}

resource "aws_ecs_cluster" "main" {
  name = "acme-${terraform.workspace}"
}
```

## Tagging

```hcl
# Tag all resources with the environment
locals {
  common_tags = {
    Environment = terraform.workspace
    ManagedBy   = "OpenTofu"
    Project     = "acme-platform"
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = local.instance_type
  tags          = local.common_tags
}
```

## Lookup Tables

```hcl
# Map workspace to configuration values
locals {
  instance_types = {
    development = "t3.micro"
    staging     = "t3.small"
    production  = "t3.xlarge"
  }

  instance_type = local.instance_types[terraform.workspace]
}
```

## Conditional Resources

```hcl
# Only create expensive resources in production
resource "aws_shield_protection" "main" {
  count    = terraform.workspace == "production" ? 1 : 0
  name     = "acme-ddos-protection"
  resource_arn = aws_alb.main.arn
}

# Create backups in staging and production, not in development
resource "aws_backup_plan" "daily" {
  count = terraform.workspace != "development" ? 1 : 0
  # ...
}
```

## Provider Configuration

```hcl
# Use different AWS accounts per workspace
locals {
  account_ids = {
    development = "111111111111"
    staging     = "222222222222"
    production  = "333333333333"
  }
}

provider "aws" {
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::${local.account_ids[terraform.workspace]}:role/TofuRole"
  }
}
```

## Workspace Validation with Error Message

```hcl
locals {
  valid_workspaces = toset(["development", "staging", "production"])

  # This errors at plan time with a clear message if workspace is unknown
  _workspace_check = contains(local.valid_workspaces, terraform.workspace) ? null : tobool("Unknown workspace: ${terraform.workspace}")
}
```

Or with a precondition:

```hcl
resource "aws_vpc" "main" {
  cidr_block = local.cidr_block

  lifecycle {
    precondition {
      condition     = contains(["development", "staging", "production"], terraform.workspace)
      error_message = "Invalid workspace: ${terraform.workspace}"
    }
  }
}
```

## Data Source Filtering

```hcl
# Find AMIs tagged for the current workspace
data "aws_ami" "app" {
  most_recent = true
  filter {
    name   = "tag:Environment"
    values = [terraform.workspace]
  }
}
```

## Conclusion

`terraform.workspace` is the key expression for workspace-aware configuration. Use it in resource names and tags for identification, in lookup maps for per-environment sizing, and in conditional `count` expressions for environment-specific resources. Always validate that `terraform.workspace` is one of the expected values using a precondition or local expression to catch misconfiguration early.
