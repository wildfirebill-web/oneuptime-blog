# How to Distribute Providers via OCI Registries in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider, OCI

Description: Learn how to distribute custom OpenTofu providers using OCI registries as an alternative to the public registry for internal or private provider distribution.

## Introduction

OpenTofu supports distributing providers via OCI (Open Container Initiative) registries. This is an OpenTofu-specific feature that lets organizations use existing container registry infrastructure - Amazon ECR, Google Artifact Registry, Azure Container Registry, or any OCI-compliant registry - to distribute private providers without operating a separate provider registry.

## Building the Provider Binary

```bash
# Build for multiple platforms

GOOS=linux  GOARCH=amd64 go build -o bin/terraform-provider-internal_linux_amd64
GOOS=linux  GOARCH=arm64 go build -o bin/terraform-provider-internal_linux_arm64
GOOS=darwin GOARCH=arm64 go build -o bin/terraform-provider-internal_darwin_arm64
```

## Packaging as an OCI Artifact

Use the `oras` CLI to push the provider binary as an OCI artifact:

```bash
# Install oras
brew install oras

# Push the provider binary as an OCI artifact
oras push registry.acme-corp.com/terraform-providers/internal:v1.0.0 \
  --artifact-type application/vnd.opentofu.provider.v1 \
  bin/terraform-provider-internal_linux_amd64:application/octet-stream

# Tag as latest
oras tag registry.acme-corp.com/terraform-providers/internal:v1.0.0 \
       registry.acme-corp.com/terraform-providers/internal:latest
```

## Configuring OpenTofu to Use an OCI Provider Registry

```hcl
# ~/.tofurc
provider_installation {
  oci_mirror {
    url     = "oci://registry.acme-corp.com/terraform-providers"
    include = ["acme-corp/internal"]
  }
  direct {
    exclude = ["acme-corp/internal"]
  }
}
```

## Declaring the Provider in Configuration

```hcl
# versions.tf
terraform {
  required_providers {
    internal = {
      source  = "acme-corp/internal"
      version = "~> 1.0"
    }
  }
}
```

## Authentication

OCI provider mirrors use the same authentication as container registries:

```bash
# Authenticate with your OCI registry
docker login registry.acme-corp.com

# For Amazon ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  123456789012.dkr.ecr.us-east-1.amazonaws.com
```

## Using Amazon ECR for Provider Distribution

```hcl
# ~/.tofurc
provider_installation {
  oci_mirror {
    url     = "oci://123456789012.dkr.ecr.us-east-1.amazonaws.com/terraform-providers"
    include = ["acme-corp/*"]
  }
  direct {
    exclude = ["acme-corp/*"]
  }
}
```

## Version Management

Push new versions with semantic version tags:

```bash
# Release v1.1.0
oras push registry.acme-corp.com/terraform-providers/internal:v1.1.0 \
  bin/terraform-provider-internal_linux_amd64:application/octet-stream

# Update the provider version constraint in your configurations
# version = "~> 1.1"
```

## Benefits Over Public Registry

- No need for a public GitHub repository
- Access controlled via registry IAM/RBAC
- Works with existing container registry infrastructure
- Supports air-gapped environments
- Immutable artifacts via digest pinning

## Conclusion

OCI registry distribution for providers is a powerful OpenTofu-exclusive feature for organizations with private providers. It reuses existing container registry infrastructure and access control, eliminating the need to operate a separate provider registry. Use it for internal providers that should not be published to the public registry.
