# How to Use Workspaces for Per-Developer Environments in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Workspaces, Developer Environments, Infrastructure as Code, Terraform

Description: Learn how to use OpenTofu workspaces to give every developer their own isolated cloud environment without duplicating configuration files.

Per-developer environments let engineers test infrastructure changes safely without interfering with shared staging or production environments. OpenTofu workspaces provide a lightweight mechanism to achieve this: a single configuration directory can manage many isolated state files, one per workspace.

## Why Per-Developer Environments?

When multiple developers share a staging environment, infrastructure changes can collide — one developer's test might destroy another's work. Per-developer environments solve this by:

- Isolating state so changes do not affect teammates.
- Reducing "it works on my machine" incidents.
- Enabling parallel feature development that touches infrastructure.

## Workspace Naming Convention

Use a consistent pattern so workspaces are easy to identify and clean up. A recommended approach is `dev-<username>`:

```bash
# Create a workspace for developer "alice"
tofu workspace new dev-alice

# Switch to Alice's workspace
tofu workspace select dev-alice

# Confirm current workspace
tofu workspace show
```

## Using terraform.workspace in Configuration

Reference `terraform.workspace` inside your configuration to parameterize resources by developer name. This avoids naming collisions between workspaces.

```hcl
# main.tf
locals {
  # Derive a short identifier from the workspace name
  env = terraform.workspace  # e.g. "dev-alice"
}

resource "aws_s3_bucket" "dev_bucket" {
  # Each developer gets their own uniquely named bucket
  bucket = "myapp-${local.env}-assets"

  tags = {
    Environment = local.env
    ManagedBy   = "opentofu"
  }
}

resource "aws_instance" "dev_server" {
  ami           = var.ami_id
  instance_type = "t3.micro"  # Small size for dev workloads

  tags = {
    Name = "dev-server-${local.env}"
  }
}
```

## Limiting Resources in Dev Workspaces

Use conditional logic to keep per-developer environments cheap by reducing instance sizes or disabling expensive features.

```hcl
# variables.tf
variable "default_instance_type" {
  default = "t3.micro"
}

locals {
  # Use t3.micro for dev workspaces, m5.large for production
  instance_type = startswith(terraform.workspace, "dev-") ? "t3.micro" : var.default_instance_type
}

resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = local.instance_type
}
```

## Workflow for Developers

A typical daily workflow using per-developer workspaces:

```bash
# 1. Navigate to the infrastructure directory
cd infra/

# 2. Select or create your workspace
tofu workspace select dev-alice || tofu workspace new dev-alice

# 3. Apply your changes
tofu apply -var-file="dev.tfvars"

# 4. Test your application changes against the new infra
# ...

# 5. Destroy when done to avoid costs
tofu destroy -var-file="dev.tfvars"
```

## Storing Workspace State Remotely

Configure a remote backend so each developer's state is stored safely and accessible to the team.

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket = "mycompany-opentofu-state"
    key    = "app/terraform.tfstate"   # workspace name is appended automatically
    region = "us-east-1"

    # Each workspace gets its own state file:
    # s3://mycompany-opentofu-state/env:/dev-alice/app/terraform.tfstate
  }
}
```

## Best Practices

- **Automate creation**: Integrate workspace creation into developer onboarding scripts.
- **Enforce naming**: Use CI checks or a wrapper script to ensure workspace names follow the `dev-<username>` pattern.
- **Set a TTL**: Add a scheduled job to list and destroy workspaces older than a defined period (see the "Clean Up Stale Workspaces" guide).
- **Use cost controls**: Tag all dev resources with the developer's name to track and limit spending.

## Conclusion

Per-developer workspaces in OpenTofu provide a simple yet powerful way to give each team member an isolated environment using a single codebase. By combining `terraform.workspace` references, conditional sizing, and remote backends, you can offer full-stack isolation without the overhead of maintaining separate configuration repositories for every developer.
