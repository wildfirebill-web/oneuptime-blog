# How the OpenTofu Provider Registry Protocol Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Registry, Protocol, Internals, Infrastructure as Code

Description: Learn how the OpenTofu provider registry protocol works - how providers are discovered, downloaded, and verified, and how to set up network mirrors for air-gapped environments.

## Introduction

When you run `tofu init`, OpenTofu uses the provider registry protocol to discover, version, and download providers. Understanding this protocol explains how provider installation works, how to set up mirrors, and how to debug `tofu init` failures.

## Provider Source Address Format

```text
[<HOSTNAME>/]<NAMESPACE>/<TYPE>

Examples:
  hashicorp/aws                              # Short form (resolved to registry.opentofu.org)
  registry.opentofu.org/hashicorp/aws        # Explicit registry
  registry.terraform.io/hashicorp/aws        # Terraform registry fallback
  my-registry.example.com/my-org/custom-provider  # Private registry
```

## Service Discovery

```bash
# OpenTofu first discovers the registry's API endpoints

curl -s https://registry.opentofu.org/.well-known/terraform.json | jq .

# Response:
{
  "modules.v1": "/v1/modules/",
  "providers.v1": "/v1/providers/",
  "login.v1": "/oauth2/..."
}
```

## Provider Versions API

```bash
# List all available versions for a provider
curl -s "https://registry.opentofu.org/v1/providers/hashicorp/aws/versions" | jq '.versions[-3:]'

# Response:
[
  {"version": "5.29.0", "protocols": ["6.0"], "platforms": [...]},
  {"version": "5.30.0", "protocols": ["6.0"], "platforms": [...]},
  {"version": "5.31.0", "protocols": ["6.0"], "platforms": [...]}
]
```

## Provider Download API

```bash
# Get download URL for a specific version and platform
curl -s "https://registry.opentofu.org/v1/providers/hashicorp/aws/5.31.0/download/linux/amd64" | jq .

# Response:
{
  "protocols": ["6.0"],
  "os": "linux",
  "arch": "amd64",
  "filename": "terraform-provider-aws_5.31.0_linux_amd64.zip",
  "download_url": "https://releases.hashicorp.com/terraform-provider-aws/5.31.0/terraform-provider-aws_5.31.0_linux_amd64.zip",
  "shasums_url": "https://releases.hashicorp.com/terraform-provider-aws/5.31.0/terraform-provider-aws_5.31.0_SHA256SUMS",
  "shasums_signature_url": "https://releases.hashicorp.com/terraform-provider-aws/5.31.0/terraform-provider-aws_5.31.0_SHA256SUMS.sig",
  "shasum": "abc123...",
  "signing_keys": {
    "gpg_public_keys": [...]
  }
}
```

## Provider Installation Directory

```bash
# Providers are cached in .terraform/providers/
ls .terraform/providers/registry.opentofu.org/hashicorp/aws/5.31.0/linux_amd64/
# terraform-provider-aws_v5.31.0_x5

# Global cache directory (shared across projects)
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
ls $TF_PLUGIN_CACHE_DIR/registry.opentofu.org/hashicorp/aws/
```

## Setting Up a Network Mirror

For air-gapped or accelerated environments:

```hcl
# ~/.terraformrc
provider_installation {
  network_mirror {
    url     = "https://my-internal-mirror.example.com/providers/"
    include = ["registry.opentofu.org/*/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
```

The network mirror must implement the provider mirror protocol:

```bash
# Mirror URL structure:
# /providers/{hostname}/{namespace}/{type}/index.json  - list versions
# /providers/{hostname}/{namespace}/{type}/{version}.json - platform downloads

# index.json
{
  "versions": {
    "5.31.0": {},
    "5.30.0": {}
  }
}

# 5.31.0.json
{
  "archives": {
    "linux_amd64": {
      "url": "terraform-provider-aws_5.31.0_linux_amd64.zip",
      "hashes": ["h1:abc123...", "zh:def456..."]
    }
  }
}
```

## Setting Up a Filesystem Mirror

```bash
# Download providers to a local directory for air-gapped environments
tofu providers mirror /opt/tofu-provider-mirror

# Directory structure:
# /opt/tofu-provider-mirror/
#   registry.opentofu.org/
#     hashicorp/
#       aws/
#         index.json
#         5.31.0.json
#         terraform-provider-aws_5.31.0_linux_amd64.zip
```

```hcl
# ~/.terraformrc - use filesystem mirror
provider_installation {
  filesystem_mirror {
    path    = "/opt/tofu-provider-mirror"
    include = ["registry.opentofu.org/*/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
```

## Lock File and Verification

After `tofu init`, the lock file records provider hashes:

```hcl
# .terraform.lock.hcl
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:hash-of-zip-contents...",
    "zh:sha256-of-zip-file...",
  ]
}
```

```bash
# Verify the lock file hasn't been tampered with
tofu init -lockfile=readonly

# Lock providers for multiple platforms
tofu providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  registry.opentofu.org/hashicorp/aws
```

## Conclusion

The OpenTofu provider registry protocol is a REST API that enables service discovery, version listing, platform-specific downloads, and cryptographic verification. For air-gapped environments, implement a network mirror or use `tofu providers mirror` to populate a filesystem mirror. The `.terraform.lock.hcl` file records provider hashes for supply chain verification - always commit it to version control and use `-lockfile=readonly` in CI to enforce it.
