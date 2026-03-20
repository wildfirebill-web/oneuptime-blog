# How to Understand the Default Workspace in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Workspaces

Description: Learn what the default workspace is in OpenTofu, how it differs from named workspaces, and when to use it.

## Introduction

Every OpenTofu configuration starts in the `default` workspace. It is the initial workspace created automatically and cannot be deleted or renamed. Understanding how the default workspace behaves differently from named workspaces helps avoid confusion about state file locations and workspace-aware expressions.

## The Default Workspace

```bash
# A fresh configuration starts in default
tofu workspace list
# * default
```

```bash
# The current workspace name
tofu workspace show
# default
```

## State File Location Differs

The default workspace uses a different state file path than named workspaces:

**Local backend:**
```
terraform.tfstate          ← default workspace
terraform.tfstate.d/
├── staging/terraform.tfstate
└── production/terraform.tfstate
```

**S3 backend:**
```
s3://bucket/prefix/terraform.tfstate           ← default workspace
s3://bucket/prefix/env:/staging/terraform.tfstate
s3://bucket/prefix/env:/production/terraform.tfstate
```

The default workspace state lives at the top-level path, not under `env:/default/`.

## Using terraform.workspace in the Default Workspace

```hcl
resource "aws_s3_bucket" "state" {
  # In the default workspace, terraform.workspace == "default"
  bucket = "acme-app-${terraform.workspace}"
  # Results in: acme-app-default
}
```

This means you should handle the `default` workspace explicitly if your naming convention doesn't include the word "default":

```hcl
locals {
  env_name = terraform.workspace == "default" ? "production" : terraform.workspace
}

resource "aws_s3_bucket" "data" {
  bucket = "acme-data-${local.env_name}"
}
```

## When the Default Workspace Is Used

Many teams use the default workspace as their only workspace (no workspace isolation at all). This is fine when:
- You manage a single environment
- You use separate configuration directories for different environments
- You have only one state file per configuration root

## Preventing Use of the Default Workspace

Some teams enforce that the default workspace is never used for deployments:

```hcl
resource "null_resource" "workspace_guard" {
  lifecycle {
    precondition {
      condition     = terraform.workspace != "default"
      error_message = "Do not deploy from the default workspace. Use 'staging' or 'production'."
    }
  }
}
```

## Checking You Are Not in Default Before Destructive Operations

```bash
#!/bin/bash
WS=$(tofu workspace show)
if [ "$WS" = "default" ]; then
  echo "Error: Do not run destroy on the default workspace."
  exit 1
fi
tofu destroy -auto-approve
```

## Switching Back to Default

```bash
# Return to the default workspace
tofu workspace select default
```

## Conclusion

The `default` workspace is the implicit starting point for all OpenTofu configurations. Its state file lives at the root path of the backend, not under `env:/default/`. Use `terraform.workspace` to detect when you are in the default workspace and handle naming accordingly. For environments that must always be explicitly named, add a precondition that rejects runs in the default workspace.
