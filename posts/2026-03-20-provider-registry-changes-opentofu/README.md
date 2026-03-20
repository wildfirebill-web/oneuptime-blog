# How OpenTofu Provider Registry Differs from Terraform

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Registry, Terraform, Migration, Infrastructure as Code

Description: Learn how OpenTofu's provider registry (registry.opentofu.org) differs from Terraform's (registry.terraform.io) — what providers are available, how to handle providers not yet mirrored, and how to configure fallback sources.

## Introduction

OpenTofu uses `registry.opentofu.org` as its default provider registry, while Terraform uses `registry.terraform.io`. Most major providers are mirrored or republished to the OpenTofu registry, but there are differences you should understand when migrating.

## What Changed

When you run `tofu init`, OpenTofu resolves providers from `registry.opentofu.org` by default. The OpenTofu registry mirrors the Terraform registry for most providers, including all HashiCorp-maintained providers.

```hcl
# This works identically in both Terraform and OpenTofu
# OpenTofu resolves from registry.opentofu.org/hashicorp/aws
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}
```

## When a Provider Is Not in the OpenTofu Registry

Some smaller or vendor-specific providers may not yet be mirrored. Use an explicit source to fall back to the Terraform registry:

```hcl
terraform {
  required_providers {
    # If vendor-provider is not on registry.opentofu.org,
    # specify the Terraform registry explicitly
    vendor-tool = {
      source  = "registry.terraform.io/vendor/vendor-tool"
      version = "~> 1.0"
    }
  }
}
```

OpenTofu supports `registry.terraform.io` as a source for providers that aren't yet in the OpenTofu registry.

## Checking Provider Availability

```bash
# Check if a provider is available in the OpenTofu registry
curl -s "https://registry.opentofu.org/v1/providers/hashicorp/aws/versions" | jq '.versions[-1].version'

# Check for a vendor-specific provider
curl -s "https://registry.opentofu.org/v1/providers/datadog/datadog/versions" | jq '.versions[-1].version'
```

## Network Mirror for Air-Gapped Environments

For environments without internet access, set up a network mirror:

```hcl
# ~/.terraformrc (also applies to OpenTofu)
provider_installation {
  network_mirror {
    url     = "https://my-internal-mirror.example.com/providers/"
    include = ["registry.opentofu.org/*/*", "registry.terraform.io/*/*"]
  }

  # Fallback to direct download if not in mirror
  direct {
    exclude = []
  }
}
```

## Filesystem Mirror

```hcl
# ~/.terraformrc
provider_installation {
  filesystem_mirror {
    path    = "/opt/opentofu-providers"
    include = ["registry.opentofu.org/*/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/*/*"]  # Don't try direct if mirror fails
  }
}
```

Populate the filesystem mirror:

```bash
# Download providers to a local directory
tofu providers mirror /opt/opentofu-providers

# Directory structure created:
# /opt/opentofu-providers/registry.opentofu.org/hashicorp/aws/5.x.x/linux_amd64/
```

## Lock File Differences

The `.terraform.lock.hcl` records provider hashes. OpenTofu's registry uses different signing keys than Terraform's, so the hashes will differ:

```hcl
# After tofu init, the lock file contains OpenTofu registry hashes
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",   # OpenTofu registry hash
    "zh:def456...",  # SHA256 zip hash
  ]
}
```

```bash
# Always regenerate the lock file when switching from terraform to tofu
rm .terraform.lock.hcl
tofu init
git add .terraform.lock.hcl
git commit -m "Regenerate provider lock file for OpenTofu"
```

## Adding Multiple Platforms to Lock File

```bash
# Lock providers for multiple platforms (for cross-platform CI)
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  registry.opentofu.org/hashicorp/aws \
  registry.opentofu.org/hashicorp/azurerm \
  registry.opentofu.org/hashicorp/google
```

## Conclusion

OpenTofu's provider registry at `registry.opentofu.org` mirrors the Terraform registry for all major providers. Most migrations require no `source` changes — `tofu init` resolves the same provider namespace from the OpenTofu registry. For providers not yet mirrored, specify `registry.terraform.io/namespace/name` explicitly. Always regenerate `.terraform.lock.hcl` after switching from `terraform` to `tofu`, as the registry signing keys differ.
