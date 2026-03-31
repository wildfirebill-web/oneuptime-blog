# How to Use Bitbucket Module Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to use Bitbucket repository module sources in OpenTofu with shorthand syntax and authentication configuration.

## Introduction

OpenTofu supports Bitbucket as a module source with a shorthand syntax similar to GitHub. This enables teams using Atlassian Bitbucket for version control to reference their infrastructure modules directly.

## Syntax

```hcl
module "example" {
  # Bitbucket shorthand
  source = "bitbucket.org/myworkspace/my-module"
  
  # With subdirectory
  source = "bitbucket.org/myworkspace/modules//vpc"
  
  # With version ref
  source = "bitbucket.org/myworkspace/modules//vpc?ref=v1.2.0"
}
```

## Examples

### Basic Bitbucket Module

```hcl
module "vpc" {
  source = "bitbucket.org/mycompany/terraform-modules//vpc?ref=v2.0.0"

  cidr_block  = "10.0.0.0/16"
  az_count    = 3
  environment = var.environment
}
```

### Multiple Modules from Same Repository

```hcl
module "vpc" {
  source = "bitbucket.org/mycompany/infra-modules//networking/vpc?ref=v1.5.0"
  cidr_block = "10.0.0.0/16"
}

module "security_groups" {
  source = "bitbucket.org/mycompany/infra-modules//networking/security-groups?ref=v1.5.0"
  vpc_id = module.vpc.vpc_id
}

module "rds" {
  source = "bitbucket.org/mycompany/infra-modules//database/rds?ref=v1.5.0"
  subnet_ids = module.vpc.private_subnet_ids
}
```

## Authentication

### App Passwords (HTTPS)

```bash
# Create an App Password in Bitbucket: Settings > Security > App passwords

export BITBUCKET_USER="myusername"
export BITBUCKET_APP_PASSWORD="your-app-password"

git config --global url."https://${BITBUCKET_USER}:${BITBUCKET_APP_PASSWORD}@bitbucket.org".insteadOf "https://bitbucket.org"
```

### SSH Keys

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "bitbucket-deploy-key"

# Add public key to Bitbucket: Account > SSH keys
cat ~/.ssh/id_ed25519.pub
```

### In CI/CD (GitHub Actions or Bitbucket Pipelines)

```yaml
# Bitbucket Pipelines example
pipelines:
  default:
    - step:
        name: Plan Infrastructure
        script:
          - git config --global url."https://x-token-auth:${BITBUCKET_TOKEN}@bitbucket.org".insteadOf "https://bitbucket.org"
          - tofu init
          - tofu plan
```

## Step-by-Step Usage

1. Create or identify your Bitbucket module repository.
2. Tag a release in Bitbucket.
3. Reference with the shorthand syntax.
4. Configure authentication (App Password or SSH).
5. Run `tofu init`.

## Conclusion

Bitbucket module sources work identically to GitHub sources in OpenTofu, with a shorthand that accepts `bitbucket.org/<workspace>/<repo>` format. Configure authentication via App Passwords for HTTPS or SSH keys for key-based access, especially in CI/CD pipelines.
