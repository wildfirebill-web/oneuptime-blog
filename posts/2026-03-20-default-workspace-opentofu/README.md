# How to Understand the Default Workspace in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Workspaces

Description: Learn about the OpenTofu default workspace, its special behavior compared to named workspaces, and when to use it versus creating dedicated workspaces.

## Introduction

Every OpenTofu configuration starts with a single workspace called `default`. Understanding its behavior - particularly how it differs from named workspaces in terms of state file paths - is important for designing a workspace strategy.

## What Is the Default Workspace?

The default workspace is:
- **Always present**: Cannot be deleted
- **The initial workspace**: New configurations start here
- **Specially named in state paths**: Uses a different path structure than named workspaces

```bash
# See the default workspace

tofu workspace list
# * default

tofu workspace show
# default
```

## Default Workspace State Paths

The default workspace stores state differently from named workspaces:

### S3 Backend

```text
S3 bucket:
├── app/terraform.tfstate           ← DEFAULT workspace state (NOT in env:/ prefix)
└── env:/
    ├── production/
    │   └── app/terraform.tfstate   ← PRODUCTION workspace state
    └── staging/
        └── app/terraform.tfstate   ← STAGING workspace state
```

The default workspace state is at the root key path, while named workspaces are nested under `env:/` (or your custom `workspace_key_prefix`).

### Local Backend

```text
project/
├── terraform.tfstate          ← DEFAULT workspace state (at root)
└── terraform.tfstate.d/
    ├── production/
    │   └── terraform.tfstate  ← PRODUCTION workspace state
    └── staging/
        └── terraform.tfstate  ← STAGING workspace state
```

## Implications for Existing State

If you have existing state in the default workspace and add workspaces later:

```bash
# Existing infrastructure is in the default workspace
tofu state list  # Shows existing resources in default workspace

# Creating a new workspace does NOT affect default workspace state
tofu workspace new production
# production workspace starts empty, default is unchanged

# Switching back to default
tofu workspace select default
tofu state list  # Still shows all original resources
```

## The terraform.workspace Value

In the default workspace, `terraform.workspace` returns `"default"`:

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"

  tags = {
    # In default workspace: "app-default"
    # In production workspace: "app-production"
    Name = "app-${terraform.workspace}"
  }
}
```

This can cause issues if your configuration uses `terraform.workspace` in resource names that must be unique - `"default"` might not be a meaningful environment name.

## When the Default Workspace Is Appropriate

Use the default workspace when:
- You're managing a single environment (no need for workspace isolation)
- You're just learning or experimenting
- Your environments are isolated via separate configurations (not workspaces)

## When to Avoid the Default Workspace

Avoid using the default workspace for production when you also have named workspaces - it can lead to confusion about which resources it manages.

```hcl
# Common mistake: checking for specific workspace names without handling default
locals {
  # This might not behave as expected in default workspace
  instance_type = {
    development = "t3.micro"
    staging     = "t3.small"
    production  = "t3.large"
    # "default" not included - will cause a lookup error!
  }
}

# Better: handle the default case
locals {
  instance_type = lookup({
    development = "t3.micro"
    staging     = "t3.small"
    production  = "t3.large"
  }, terraform.workspace, "t3.micro")  # Default fallback
}
```

## Checking Whether You're in Default

```hcl
# Sometimes useful to warn against running in default workspace
resource "null_resource" "workspace_check" {
  triggers = {
    workspace = terraform.workspace
  }

  provisioner "local-exec" {
    command = terraform.workspace == "default" ? "echo 'WARNING: Running in default workspace'" : "echo 'OK: Running in ${terraform.workspace}'"
  }
}
```

## Conclusion

The default workspace is the starting point for every OpenTofu configuration and has special state path behavior compared to named workspaces. For single-environment setups, the default workspace is perfectly appropriate. For multi-environment setups with workspaces, create named workspaces for each environment and either repurpose the default or avoid using it for production workloads to prevent confusion about which environment it represents.
