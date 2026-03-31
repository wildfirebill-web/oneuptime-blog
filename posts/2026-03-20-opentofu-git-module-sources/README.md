# How to Use Git Repository Module Sources in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Infrastructure as Code, Terraform, IaC, DevOps, Module

Description: Learn how to reference modules stored in Git repositories in OpenTofu using HTTPS and SSH URLs with version pinning via ref parameter.

## Introduction

Git repository module sources allow you to reference OpenTofu modules stored in any Git repository - public or private. This is ideal for sharing modules across teams and repositories with proper version control via Git tags and refs.

## Syntax

```hcl
module "example" {
  # HTTPS
  source = "git::https://github.com/myorg/terraform-modules.git//modules/vpc?ref=v1.2.0"
  
  # SSH
  source = "git::git@github.com:myorg/terraform-modules.git//modules/vpc?ref=v1.2.0"
  
  # Without subdir (entire repo is the module)
  source = "git::https://github.com/myorg/my-module.git?ref=v1.0.0"
}
```

- `git::` prefix specifies Git source
- `//` separates the repo URL from the subdirectory path
- `?ref=` specifies the Git ref (tag, branch, or commit hash)

## Examples

### Pinning to a Version Tag

```hcl
module "vpc" {
  source = "git::https://github.com/myorg/terraform-aws-modules.git//modules/vpc?ref=v2.1.0"

  cidr_block   = "10.0.0.0/16"
  az_count     = 3
  environment  = var.environment
}
```

### Pinning to a Specific Commit

```hcl
module "security_groups" {
  # Pinning to exact commit for maximum reproducibility
  source = "git::https://github.com/myorg/infra-modules.git//security-groups?ref=a1b2c3d4e5f6"

  vpc_id      = module.vpc.vpc_id
  environment = var.environment
}
```

### Using a Branch (Not Recommended for Production)

```hcl
module "experimental" {
  # Branches change over time - use tags for production
  source = "git::https://github.com/myorg/modules.git//new-feature?ref=feature/my-feature"

  name = "test"
}
```

### SSH for Private Repositories

```hcl
module "internal_module" {
  source = "git::git@github.com:mycompany/private-modules.git//networking?ref=v3.0.0"

  vpc_cidr = "10.100.0.0/16"
}
```

## Authentication

### HTTPS with Token

Set the `GIT_ASKPASS` or `GIT_TERMINAL_PROMPT=0` environment variables, or configure Git credentials:

```bash
git config --global credential.helper store
# or

export GITHUB_TOKEN=ghp_xxxxx
```

### SSH Keys

Configure SSH key agent:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### In CI/CD

Use SSH deploy keys or GitHub token with git credential helper:

```bash
# In CI environment
git config --global url."https://oauth2:${GITHUB_TOKEN}@github.com".insteadOf "https://github.com"
```

## Step-by-Step Usage

1. Tag your module repository: `git tag v1.0.0 && git push origin v1.0.0`
2. Reference with `?ref=v1.0.0` in your module source.
3. Run `tofu init` to clone the module.

```bash
tofu init
# Downloads and caches the module

tofu plan
```

## Best Practices

- Always pin to a specific tag or commit hash in production
- Never use `main` or `master` branch as a module source in production
- Use semantic versioning (v1.2.3) for module tags
- Consider a private module registry for internal modules at scale

## Conclusion

Git repository module sources enable cross-repository module sharing in OpenTofu. By using `?ref=` to pin to specific tags or commits, you get reproducible, versioned module references. This approach is ideal for platform teams that maintain shared infrastructure modules used by multiple product teams.
