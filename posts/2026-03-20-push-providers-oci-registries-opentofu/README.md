# How to Push Providers to OCI Registries with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, OCI Registry, Provider Distribution, Container, Infrastructure

Description: Learn how to push OpenTofu provider plugins to OCI-compatible registries for distribution and version management, leveraging container registry infrastructure you already have.

## Introduction

OpenTofu 1.8+ supports using OCI (Open Container Initiative) registries as a source for provider plugins. This lets you store providers alongside your container images in the same registry infrastructure - ECR, ACR, GCR, or any OCI-compliant registry. Pushing providers to OCI is different from pushing container images: you're packaging provider binaries as OCI artifacts, not Docker images.

## Prerequisites

```bash
# Install oras (OCI Registry AS Storage) for pushing OCI artifacts

# macOS
brew install oras

# Linux
curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
tar -xzf oras_1.1.0_linux_amd64.tar.gz
sudo mv oras /usr/local/bin/

oras version
```

## Provider OCI Package Structure

```bash
# OpenTofu OCI providers use a specific manifest structure
# Each provider version is pushed as an OCI artifact with:
# - manifest.json: OCI image manifest
# - provider binary zip files as layers
# - index.json metadata

# The expected directory layout before pushing:
provider-package/
├── terraform-provider-myprovider_1.0.0_linux_amd64.zip
├── terraform-provider-myprovider_1.0.0_linux_arm64.zip
├── terraform-provider-myprovider_1.0.0_darwin_arm64.zip
├── terraform-provider-myprovider_1.0.0_SHA256SUMS
└── terraform-provider-myprovider_1.0.0_SHA256SUMS.sig
```

## Building a Provider for OCI Distribution

```makefile
# Makefile for building a custom provider

PROVIDER_NAME = myprovider
NAMESPACE = mycompany
VERSION = 1.0.0
REGISTRY = registry.internal.company.com

.PHONY: build package push

build:
    @for OS in linux darwin windows; do \
        for ARCH in amd64 arm64; do \
            echo "Building $${OS}_$${ARCH}..."; \
            GOOS=$${OS} GOARCH=$${ARCH} go build \
                -o dist/terraform-provider-$(PROVIDER_NAME)_$(VERSION)_$${OS}_$${ARCH}; \
        done; \
    done

package: build
    @mkdir -p dist
    @for OS in linux darwin windows; do \
        for ARCH in amd64 arm64; do \
            zip -j dist/terraform-provider-$(PROVIDER_NAME)_$(VERSION)_$${OS}_$${ARCH}.zip \
                dist/terraform-provider-$(PROVIDER_NAME)_$(VERSION)_$${OS}_$${ARCH}; \
        done; \
    done
    @cd dist && sha256sum *.zip > terraform-provider-$(PROVIDER_NAME)_$(VERSION)_SHA256SUMS
    @gpg --detach-sign dist/terraform-provider-$(PROVIDER_NAME)_$(VERSION)_SHA256SUMS

push: package
    $(MAKE) push-to-oci
```

## Pushing to OCI Registry with oras

```bash
#!/bin/bash
# push-provider-oci.sh

set -euo pipefail

PROVIDER_NAME="myprovider"
NAMESPACE="mycompany"
VERSION="1.0.0"
REGISTRY="registry.internal.company.com"
DIST_DIR="./dist"

# Login to registry
oras login "$REGISTRY" \
  --username "$REGISTRY_USER" \
  --password "$REGISTRY_PASSWORD"

# Construct the OCI reference
OCI_REF="${REGISTRY}/${NAMESPACE}/${PROVIDER_NAME}:${VERSION}"

echo "Pushing provider to: $OCI_REF"

# Push all platform-specific zips as a multi-platform artifact
cd "$DIST_DIR"

oras push "$OCI_REF" \
  --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
  terraform-provider-${PROVIDER_NAME}_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64 \
  terraform-provider-${PROVIDER_NAME}_${VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64 \
  terraform-provider-${PROVIDER_NAME}_${VERSION}_darwin_arm64.zip:application/vnd.opentofu.provider.v1.darwin.arm64 \
  terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums \
  terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS.sig:application/vnd.opentofu.provider.v1.shasums.sig

echo "Provider pushed successfully: $OCI_REF"

# Also tag as latest
oras tag "${REGISTRY}" "${NAMESPACE}/${PROVIDER_NAME}:${VERSION}" "${NAMESPACE}/${PROVIDER_NAME}:latest"
```

## Configuring OpenTofu to Pull from OCI

```hcl
# terraform.rc - configure OCI provider source
provider_installation {
  oci_mirror {
    url     = "oci://registry.internal.company.com"
    include = ["registry.opentofu.org/mycompany/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/mycompany/*"]
  }
}
```

```hcl
# In your OpenTofu configuration
terraform {
  required_providers {
    myprovider = {
      source  = "mycompany/myprovider"
      version = "~> 1.0"
    }
  }
}
```

## GitHub Actions Pipeline for Provider Publishing

```yaml
# .github/workflows/publish-provider-oci.yml
name: Publish Provider to OCI

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Build provider
        run: make package VERSION=${{ steps.version.outputs.version }}

      - name: Install oras
        run: |
          curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
          tar -xzf oras_1.1.0_linux_amd64.tar.gz
          sudo mv oras /usr/local/bin/

      - name: Login to registry
        run: |
          oras login ghcr.io \
            --username ${{ github.actor }} \
            --password ${{ secrets.GITHUB_TOKEN }}

      - name: Push to OCI registry
        run: |
          VERSION=${{ steps.version.outputs.version }}
          REGISTRY="ghcr.io/${{ github.repository_owner }}"

          cd dist/
          oras push "${REGISTRY}/provider-myprovider:${VERSION}" \
            terraform-provider-myprovider_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64 \
            terraform-provider-myprovider_${VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64 \
            terraform-provider-myprovider_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums
```

## Inspecting OCI Provider Artifacts

```bash
# Inspect what's stored in the OCI registry
oras manifest fetch registry.internal.company.com/mycompany/myprovider:1.0.0

# List tags
oras repo tags registry.internal.company.com/mycompany/myprovider

# Pull and verify locally
oras pull registry.internal.company.com/mycompany/myprovider:1.0.0 \
  --output /tmp/provider-check/
ls -la /tmp/provider-check/
```

## Conclusion

Pushing OpenTofu providers to OCI registries uses `oras` to package provider binaries as OCI artifacts with content-type annotations that OpenTofu understands. The `oci_mirror` configuration in `terraform.rc` tells OpenTofu to pull providers from the OCI registry instead of the Terraform registry. This approach is particularly useful for organizations already running OCI-compliant registries (ECR, ACR, GCR, GHCR), as it eliminates the need to run a separate provider registry server.
