# How to Use GitHub Module Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to use the shorthand GitHub module source syntax in OpenTofu for cleaner, more readable module references.

## Introduction

OpenTofu supports a shorthand syntax for GitHub module sources that is more readable than the full `git::https://` URL. Instead of writing the full Git URL, you can use `github.com/<owner>/<repo>` format.

## Syntax

```hcl
module "example" {
  # Shorthand GitHub syntax (no git:: prefix needed)
  source = "github.com/myorg/terraform-aws-vpc"
  
  # With subdirectory
  source = "github.com/myorg/terraform-modules//vpc"
  
  # With version ref
  source = "github.com/myorg/terraform-aws-vpc?ref=v2.1.0"
}
```

## Examples

### Public GitHub Repository

```hcl
module "vpc" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc?ref=v5.0.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false
}
```

### GitHub with Subdirectory

```hcl
# When a repo contains multiple modules in subdirectories

module "security_groups" {
  source = "github.com/myorg/aws-modules//security-groups?ref=v1.5.0"

  vpc_id      = aws_vpc.main.id
  environment = var.environment
}

module "iam_roles" {
  source = "github.com/myorg/aws-modules//iam/roles?ref=v1.5.0"

  environment = var.environment
}
```

### Private GitHub Repository

```hcl
module "internal_vpc" {
  # Private repo - requires authentication
  source = "github.com/mycompany/private-infra-modules//networking/vpc?ref=v3.2.1"

  cidr_block   = "10.200.0.0/16"
  az_count     = 3
  environment  = var.environment
}
```

## Authentication for Private Repos

### Personal Access Token (HTTPS)

```bash
# Set GITHUB_TOKEN environment variable
export GITHUB_TOKEN=ghp_your_token_here

# Configure Git to use it
git config --global url."https://${GITHUB_TOKEN}@github.com".insteadOf "https://github.com"
```

### SSH Key (Recommended for CI)

```bash
# Generate deploy key for the private repo
ssh-keygen -t ed25519 -C "deploy-key" -f ~/.ssh/deploy_key

# Add public key to GitHub repo Settings > Deploy Keys
cat ~/.ssh/deploy_key.pub

# Configure SSH to use the key
cat >> ~/.ssh/config << EOF
Host github.com
  IdentityFile ~/.ssh/deploy_key
  User git
EOF
```

## GitHub vs Full Git URL

```hcl
# These are equivalent:
source = "github.com/myorg/my-module?ref=v1.0.0"
source = "git::https://github.com/myorg/my-module.git?ref=v1.0.0"
```

The shorthand `github.com/...` syntax is cleaner and preferred.

## Step-by-Step Usage

1. Find or create a GitHub repository with your module code.
2. Tag a release: `git tag v1.0.0 && git push origin v1.0.0`.
3. Reference the module with the shorthand syntax.
4. Run `tofu init`.

## Conclusion

The shorthand GitHub syntax in OpenTofu makes module references cleaner and more readable than full Git URLs. Use it with `?ref=version` tags for pinned, reproducible module sources. For private repositories, configure Git authentication via HTTPS tokens or SSH keys before running `tofu init`.
