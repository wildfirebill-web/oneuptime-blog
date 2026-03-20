# How to Pull Modules from OCI Registries with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, OCI Registry, Module Source, Containers, Infrastructure

Description: Learn how to configure OpenTofu to pull modules directly from OCI-compatible registries using the oci:: source prefix for version-controlled module distribution.

## Introduction

OpenTofu 1.8+ supports the `oci::` source prefix for modules, allowing modules to be pulled directly from OCI registries. This enables teams to distribute modules through existing container registry infrastructure without running a separate module registry server. Authentication reuses Docker credential helpers, making it easy to integrate with private registries.

## Basic OCI Module Source

```hcl
# Reference a module stored in an OCI registry
module "vpc" {
  source  = "oci://registry.internal.company.com/mycompany/module-vpc:1.2.0"

  name            = "production"
  cidr            = "10.0.0.0/16"
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}
```

## Authentication for Private OCI Registries

```bash
# For GitHub Container Registry (GHCR)
echo "$GITHUB_TOKEN" | docker login ghcr.io -u "$GITHUB_ACTOR" --password-stdin

# For AWS ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789.dkr.ecr.us-east-1.amazonaws.com

# For Azure Container Registry
az acr login --name mycompanyregistry

# OpenTofu uses Docker's credential store for OCI authentication
# Configure ~/.docker/config.json or credential helpers
```

```hcl
# ~/.terraform.rc - explicit credentials for OCI registry
credentials "registry.internal.company.com" {
  token = "your-registry-token"
}
```

## Version Pinning and Constraints

```hcl
# Pin to exact version (recommended for production)
module "database" {
  source = "oci://ghcr.io/myorg/module-rds:3.1.2"

  identifier     = "production-db"
  engine         = "postgres"
  instance_class = "db.t4g.medium"
}

# Use a major version tag (set up by your push workflow)
module "security_groups" {
  source = "oci://ghcr.io/myorg/module-security-groups:2"

  vpc_id = module.vpc.vpc_id
}

# Note: OCI sources don't support version constraints like "~> 1.0"
# You must reference a specific tag or digest
```

## Using Image Digests for Immutable References

```hcl
# Reference by digest for fully immutable module references
# (immune to tag mutation)

module "vpc" {
  source = "oci://registry.internal.company.com/mycompany/module-vpc@sha256:abc123def456..."

  name = "production"
  cidr = "10.0.0.0/16"
}
```

```bash
# Get the digest of a specific tag
oras manifest fetch --descriptor \
  registry.internal.company.com/mycompany/module-vpc:1.2.0 | \
  jq -r '.digest'
# Output: sha256:abc123def456...
```

## Pulling from GitHub Container Registry

```hcl
# Public modules from GHCR (no authentication needed)
module "vpc" {
  source = "oci://ghcr.io/myorg/module-vpc:1.0.0"

  name = "production"
  cidr = "10.0.0.0/16"
}

# Private modules from GHCR (requires GITHUB_TOKEN)
module "internal_baseline" {
  source = "oci://ghcr.io/mycompany/module-account-baseline:2.1.0"
}
```

```bash
# Set GITHUB_TOKEN for private GHCR access
export GITHUB_TOKEN="ghp_yourPersonalAccessToken"
# Docker credential helper picks this up automatically
echo "$GITHUB_TOKEN" | docker login ghcr.io -u myuser --password-stdin
```

## Module Version Discovery

```bash
# List all available tags for a module in OCI
oras repo tags registry.internal.company.com/mycompany/module-vpc

# Output:
# 1
# 1.0
# 1.0.0
# 1.1
# 1.1.0
# 1.2
# 1.2.0
# latest

# Inspect module contents before using
mkdir -p /tmp/module-preview
oras pull registry.internal.company.com/mycompany/module-vpc:1.2.0 \
  --output /tmp/module-preview/
tar -tzf /tmp/module-preview/*.tgz
```

## CI/CD Pipeline Usage

```yaml
# .github/workflows/deploy.yml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      packages: read

    steps:
      - uses: actions/checkout@v4

      - name: Login to GHCR for module access
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - uses: opentofu/setup-opentofu@v1

      - name: Tofu Init (pulls OCI modules)
        run: tofu init

      - name: Tofu Plan
        run: tofu plan -out=plan.tfplan
```

## Caching OCI Modules in CI

```bash
# Cache pulled modules to avoid repeated OCI pulls
# OpenTofu caches modules in .terraform/modules/ after init

# GitHub Actions cache
- name: Cache OpenTofu modules
  uses: actions/cache@v4
  with:
    path: .terraform/modules
    key: tf-modules-${{ hashFiles('**/*.tf') }}
    restore-keys: |
      tf-modules-

- name: OpenTofu Init
  run: tofu init
  # On cache hit, modules are already in .terraform/modules/
  # tofu init will skip OCI pulls for cached modules
```

## Mixing OCI and Registry Module Sources

```hcl
# Mix OCI modules with registry and local modules
module "vpc" {
  # Internal module from OCI
  source = "oci://registry.internal.company.com/mycompany/module-vpc:2.0.0"
  name   = "production"
  cidr   = "10.0.0.0/16"
}

module "eks" {
  # Public registry module
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  cluster_name = "production"
}

module "custom_app" {
  # Local module
  source = "./modules/custom-app"
}
```

## Conclusion

Pulling modules from OCI registries uses the `oci://` source prefix with a registry URL, repository path, and tag or digest. Authentication reuses Docker's credential helpers, so any registry you can `docker login` to works with OpenTofu. Cache the `.terraform/modules/` directory in CI pipelines to avoid repeated OCI pulls. Use image digests instead of tags for production deployments where immutability matters — a digest always refers to the exact same artifact, even if the tag is later updated.
