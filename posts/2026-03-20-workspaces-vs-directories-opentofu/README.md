# How to Choose Between Workspaces and Separate Directories in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Workspaces, Directory Structure, Environments, Infrastructure as Code, Best Practices

Description: Learn when to use OpenTofu workspaces versus separate directories for managing multiple environments, and why separate directories are recommended for production environments.

## Introduction

OpenTofu workspaces and separate directories both allow you to manage multiple environments (dev, staging, prod) from the same codebase. They make different tradeoffs, and the OpenTofu community broadly recommends separate directories for production environments. Understanding why helps you make the right choice.

## OpenTofu Workspaces

Workspaces create separate state files within the same backend configuration:

```bash
# Create and switch workspaces

tofu workspace new dev
tofu workspace new staging
tofu workspace new prod

# Switch to a workspace
tofu workspace select prod

# List workspaces
tofu workspace list
# * prod
#   staging
#   dev
```

Use `terraform.workspace` in configuration:

```hcl
locals {
  instance_type = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "m5.large"
  }[terraform.workspace]
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
}
```

### Workspace Drawbacks

```text
- Single configuration for all environments - risky
- Easy to accidentally apply to wrong workspace
- No per-environment variable files
- All environments share the same code version
- Production and dev state are in the same backend bucket path
```

## Separate Directories (Recommended for Production)

Each environment is an independent configuration:

```text
environments/
├── dev/
│   ├── main.tf
│   ├── terraform.tfvars   # dev-specific values
│   └── backend.tf
├── staging/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf
└── prod/
    ├── main.tf
    ├── terraform.tfvars   # prod-specific values (no secrets)
    └── backend.tf
```

### Directory Advantages

```text
+ Completely isolated state files
+ Different variable values per environment
+ Different module versions per environment (prod can lag)
+ Independent blast radius - prod apply can't affect dev
+ Clear ownership - CODEOWNERS per environment directory
+ Environment-specific CI rules (prod requires extra approvals)
```

## When to Use Workspaces

Workspaces are appropriate for:

```hcl
# Ephemeral per-developer environments
# Feature branch environments that are destroyed when the branch is merged

locals {
  # Use workspace name as the environment identifier
  env_name = terraform.workspace  # "feature-abc123", "john-dev"
}

resource "aws_instance" "dev_box" {
  tags = { Environment = terraform.workspace }
}
```

## When to Use Separate Directories

- Any long-lived environment (dev, staging, prod)
- Environments with different risk profiles
- Environments owned by different teams
- Production infrastructure that must be protected from developer mistakes

## Recommendation

```text
Use workspaces for:    Ephemeral environments, feature branches, per-developer sandboxes
Use directories for:   Dev, staging, prod and any other persistent environment
```

## Conclusion

The OpenTofu best practice is to use separate directories for production and other long-lived environments. Workspaces are a useful tool for ephemeral environments like feature branch deployments. The extra ceremony of separate directories - maintaining multiple files - is justified by the isolation and safety it provides for production infrastructure.
