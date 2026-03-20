# How to Use Terramate with OpenTofu for Stack Management

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terramate, Stack Management, Infrastructure as Code, GitOps, DevOps

Description: Learn how to use Terramate to organize OpenTofu configurations into stacks, detect changed stacks, and orchestrate apply operations across a large monorepo.

## Introduction

Terramate is an orchestration tool for OpenTofu and Terraform that introduces the concept of "stacks" — self-contained units of infrastructure. Its killer feature is change detection: when you open a pull request, Terramate only runs `tofu plan` on the stacks that were actually modified, making large monorepos practical.

## Installing Terramate

```bash
# macOS
brew install terramate

# Linux
curl -LO https://github.com/terramate-io/terramate/releases/latest/download/terramate_linux_amd64.tar.gz
tar -xzf terramate_linux_amd64.tar.gz && sudo mv terramate /usr/local/bin/

# Verify
terramate --version
```

## Initializing a Terramate Project

```bash
# Create the root Terramate config
cat > terramate.tm.hcl <<'EOF'
terramate {
  config {
    # Tell Terramate which binary to use
    run {
      env {
        TF_CLI_ARGS = ""
      }
    }
  }
}
EOF
```

## Configuring OpenTofu as the Binary

```hcl
# terramate.tm.hcl (root)
terramate {
  config {
    # Override the terraform binary with tofu
    terraform {
      tofu_binary = "tofu"
    }
  }
}
```

## Creating Stacks

Each stack is a directory with a `stack.tm.hcl` file:

```bash
# Create stacks for different environments/components
terramate create stacks/networking
terramate create stacks/eks-cluster
terramate create stacks/rds
```

This generates a `stack.tm.hcl` in each directory:

```hcl
# stacks/networking/stack.tm.hcl — auto-generated
stack {
  name        = "networking"
  description = "VPC, subnets, and routing"
  id          = "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## Generating Shared Configuration

Terramate's code generation injects common config (like backend or provider) into every stack:

```hcl
# terramate.tm.hcl (root) — generate backend.tf in all stacks
generate_hcl "backend.tf" {
  content {
    terraform {
      backend "s3" {
        bucket         = "my-opentofu-state"
        key            = "${terramate.stack.path}/tofu.tfstate"
        region         = "us-east-1"
        dynamodb_table = "opentofu-locks"
        encrypt        = true
      }
    }
  }
}
```

After changing the generation config, regenerate files:

```bash
terramate generate
```

## Detecting Changed Stacks

Terramate compares the current branch to the default branch to find stacks with modified files:

```bash
# List stacks changed compared to main
terramate list --changed

# Run tofu plan only on changed stacks
terramate run --changed -- tofu plan

# Run tofu apply on changed stacks
terramate run --changed -- tofu apply -auto-approve
```

## CI/CD Integration

```yaml
# .github/workflows/opentofu.yml
name: OpenTofu Changed Stacks

on:
  pull_request:

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # Required for change detection

      - name: Install OpenTofu
        run: |
          curl -LO https://github.com/opentofu/opentofu/releases/download/v1.9.0/tofu_1.9.0_linux_amd64.zip
          unzip tofu_1.9.0_linux_amd64.zip && sudo mv tofu /usr/local/bin/

      - name: Install Terramate
        run: brew install terramate

      - name: Plan changed stacks
        run: terramate run --changed -- tofu plan
```

## Conclusion

Terramate makes large-scale OpenTofu monorepos manageable by treating each component as an isolated stack and only operating on what changed. Combined with code generation for shared backend and provider config, it provides a clean alternative to Terragrunt for teams who prefer Terramate's stack-centric model.
