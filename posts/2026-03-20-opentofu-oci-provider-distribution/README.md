# Distributing OpenTofu Providers via OCI Registries

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Provider, OCI, Registry

Description: Learn how OpenTofu supports OCI (Open Container Initiative) registries for distributing providers and modules.

OpenTofu introduces support for OCI (Open Container Initiative) registries as an alternative distribution channel for providers and modules. This allows organizations to use existing container registry infrastructure (Docker Hub, ECR, GCR, Harbor) to distribute OpenTofu content.

## What Is OCI Distribution?

OCI is the standard format used by container images. OpenTofu can use OCI registries to:
- Distribute providers to air-gapped environments
- Use existing registry infrastructure
- Leverage fine-grained access control from container registries
- Integrate with standard CI/CD container workflows

## Configuring OCI Provider Sources

```hcl
terraform {
  required_providers {
    # Reference a provider from an OCI registry
    mycustom = {
      source  = "oci://registry.example.com/terraform-providers/mycustom"
      version = "~> 1.0"
    }
  }
}
```

## OCI Registry Authentication

```hcl
# ~/.tofurc

provider_installation {
  oci {
    # Use credentials from Docker config
  }

  direct {}
}
```

```bash
# Authenticate with the OCI registry (uses standard Docker credentials)
docker login registry.example.com
# or
echo "$REGISTRY_PASSWORD" | docker login registry.example.com -u "$REGISTRY_USERNAME" --password-stdin

# OpenTofu reads credentials from:
# ~/.docker/config.json (Docker credentials)
# Environment variables (DOCKER_USERNAME, DOCKER_PASSWORD)
# Platform-specific credential helpers
```

## Pushing a Provider to an OCI Registry

```bash
# After building your custom provider binary
# Package it as an OCI artifact

# Install oras (OCI Registry AS Storage)
brew install oras  # macOS
# or download from https://oras.land

# Push provider binary as OCI artifact
oras push registry.example.com/terraform-providers/myprovider:1.0.0 \
  terraform-provider-myprovider_1.0.0_linux_amd64.zip:application/zip \
  terraform-provider-myprovider_1.0.0_darwin_arm64.zip:application/zip \
  --annotation "org.opentofu.provider.version=1.0.0" \
  --annotation "org.opentofu.provider.protocols=6.0"
```

## Using OCI Modules

```hcl
# Reference a module stored in an OCI registry
module "vpc" {
  source  = "oci://registry.example.com/terraform-modules/vpc:v2.1.0"

  cidr_block  = "10.0.0.0/16"
  environment = "production"
}
```

## Pushing a Module to OCI

```bash
# Package your module
tar -czf vpc-module.tar.gz ./modules/vpc/

# Push to OCI registry
oras push registry.example.com/terraform-modules/vpc:v2.1.0 \
  vpc-module.tar.gz:application/vnd.opentofu.module.v1+tar+gzip

# Tag as latest
oras tag registry.example.com/terraform-modules/vpc:v2.1.0 \
  registry.example.com/terraform-modules/vpc:latest
```

## Corporate Registry Setup (Harbor)

```yaml
# harbor-values.yaml (Helm)
expose:
  type: ingress
  ingress:
    hosts:
      core: registry.example.com

externalURL: https://registry.example.com

# Create projects for Terraform content
# Project: terraform-providers
# Project: terraform-modules
```

```hcl
# ~/.tofurc - configure OCI for corporate registry
provider_installation {
  oci {
    # Harbor uses Docker credentials
  }

  direct {
    exclude = ["registry.example.com/*/*"]
  }
}
```

```hcl
terraform {
  required_providers {
    internal = {
      source  = "oci://registry.example.com/terraform-providers/internal"
      version = "~> 2.0"
    }
  }
}
```

## Private Provider Distribution Workflow

```bash
#!/bin/bash
# ci-publish-provider.sh

VERSION=$1
REGISTRY="registry.example.com/terraform-providers"
PROVIDER="myprovider"

# Build for multiple platforms
GOOS=linux GOARCH=amd64 go build -o "terraform-provider-${PROVIDER}_linux_amd64"
GOOS=darwin GOARCH=arm64 go build -o "terraform-provider-${PROVIDER}_darwin_arm64"

# Zip binaries
zip "terraform-provider-${PROVIDER}_${VERSION}_linux_amd64.zip" \
  "terraform-provider-${PROVIDER}_linux_amd64"
zip "terraform-provider-${PROVIDER}_${VERSION}_darwin_arm64.zip" \
  "terraform-provider-${PROVIDER}_darwin_arm64"

# Push to OCI registry
oras push "${REGISTRY}/${PROVIDER}:${VERSION}" \
  "terraform-provider-${PROVIDER}_${VERSION}_linux_amd64.zip:application/zip" \
  "terraform-provider-${PROVIDER}_${VERSION}_darwin_arm64.zip:application/zip"

echo "Published ${PROVIDER} v${VERSION} to OCI registry"
```

## Advantages Over Traditional Distribution

```bash
Traditional (registry.opentofu.org):
- Public only (or requires Terraform Cloud for private)
- Must match specific naming conventions
- Registry-specific workflow

OCI Distribution:
- Works with any OCI-compatible registry
- Use existing Docker infrastructure
- Fine-grained access control (same as container images)
- Works in air-gapped environments
- Standard tooling (docker, oras, skopeo)
- Integrates with existing CI/CD for containers
```

## Conclusion

OCI distribution brings the flexibility of container registries to OpenTofu provider and module distribution. For organizations already using container registries like Harbor, ECR, or GCR, this provides a natural way to distribute private providers without needing a separate Terraform registry. It's particularly valuable for air-gapped environments and organizations with strict provenance requirements.
