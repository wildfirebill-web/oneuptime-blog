# How to Push Modules to OCI Registries with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, OCI Registry, Module Distribution, Containers, Infrastructure

Description: Learn how to package and push OpenTofu modules to OCI-compatible registries for version-controlled module distribution within your organization.

## Introduction

OpenTofu supports sourcing modules from OCI registries using the `oci::` source prefix. Pushing modules to OCI registries packages your module directories as OCI artifacts, enabling version-controlled module distribution through the same container registry infrastructure you already use for Docker images.

## Packaging a Module for OCI

```bash
# Module structure to be packaged
my-vpc-module/
├── main.tf
├── variables.tf
├── outputs.tf
├── versions.tf
└── README.md

# Create a module archive
tar -czf my-vpc-module-1.0.0.tgz \
  --exclude='.terraform' \
  --exclude='*.tfstate' \
  --exclude='*.tfstate.backup' \
  --exclude='.git' \
  -C my-vpc-module/ .

# Verify archive contents
tar -tzf my-vpc-module-1.0.0.tgz
```

## Pushing to OCI with oras

```bash
# Install oras
brew install oras  # macOS
# or: https://github.com/oras-project/oras/releases

# Login to registry
oras login registry.internal.company.com \
  --username "$REGISTRY_USER" \
  --password "$REGISTRY_PASSWORD"

# Push module as OCI artifact
REGISTRY="registry.internal.company.com"
NAMESPACE="mycompany"
MODULE="vpc"
VERSION="1.0.0"

oras push "${REGISTRY}/${NAMESPACE}/module-${MODULE}:${VERSION}" \
  --config /dev/null:application/vnd.opentofu.module.config.v1+json \
  my-vpc-module-${VERSION}.tgz:application/vnd.opentofu.module.v1.tar+gzip

echo "Module pushed: ${REGISTRY}/${NAMESPACE}/module-${MODULE}:${VERSION}"

# Tag as latest
oras tag "${REGISTRY}" \
  "${NAMESPACE}/module-${MODULE}:${VERSION}" \
  "${NAMESPACE}/module-${MODULE}:latest"
```

## Automated Push Script

```bash
#!/bin/bash
# push-module.sh - Push an OpenTofu module to OCI

set -euo pipefail

MODULE_DIR="${1:?Usage: $0 <module-dir> <version>}"
VERSION="${2:?Usage: $0 <module-dir> <version>}"
REGISTRY="${REGISTRY:-registry.internal.company.com}"
NAMESPACE="${NAMESPACE:-mycompany}"

MODULE_NAME=$(basename "$MODULE_DIR")
ARCHIVE="${MODULE_NAME}-${VERSION}.tgz"
OCI_REF="${REGISTRY}/${NAMESPACE}/module-${MODULE_NAME}:${VERSION}"

echo "Packaging module: $MODULE_DIR"
tar -czf "/tmp/$ARCHIVE" \
  --exclude='.terraform' \
  --exclude='*.tfstate*' \
  --exclude='.git' \
  -C "$MODULE_DIR" .

echo "Pushing to: $OCI_REF"
oras push "$OCI_REF" \
  --config /dev/null:application/vnd.opentofu.module.config.v1+json \
  "/tmp/$ARCHIVE:application/vnd.opentofu.module.v1.tar+gzip"

# Add semantic version tags
MAJOR=$(echo "$VERSION" | cut -d. -f1)
MINOR=$(echo "$VERSION" | cut -d. -f1-2)

oras tag "${REGISTRY}" "${NAMESPACE}/module-${MODULE_NAME}:${VERSION}" "${NAMESPACE}/module-${MODULE_NAME}:${MAJOR}"
oras tag "${REGISTRY}" "${NAMESPACE}/module-${MODULE_NAME}:${VERSION}" "${NAMESPACE}/module-${MODULE_NAME}:${MINOR}"
oras tag "${REGISTRY}" "${NAMESPACE}/module-${MODULE_NAME}:${VERSION}" "${NAMESPACE}/module-${MODULE_NAME}:latest"

rm -f "/tmp/$ARCHIVE"
echo "Done: $OCI_REF"
```

## GitHub Actions Publishing Pipeline

```yaml
# .github/workflows/publish-module.yml
name: Publish Module to OCI

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Install oras
        run: |
          curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
          tar -xzf oras_1.1.0_linux_amd64.tar.gz
          sudo mv oras /usr/local/bin/

      - name: Validate module
        run: |
          curl -Lo /usr/local/bin/tofu.zip \
            https://github.com/opentofu/opentofu/releases/download/v1.7.0/opentofu_1.7.0_linux_amd64.zip
          unzip /usr/local/bin/tofu.zip -d /usr/local/bin/
          tofu init -backend=false
          tofu validate

      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Login to GitHub Container Registry
        run: |
          oras login ghcr.io \
            --username ${{ github.actor }} \
            --password ${{ secrets.GITHUB_TOKEN }}

      - name: Package and push module
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          MODULE_NAME="${{ github.event.repository.name }}"
          ORG="${{ github.repository_owner }}"

          # Create archive
          tar -czf "${MODULE_NAME}-${VERSION}.tgz" \
            --exclude='.terraform' \
            --exclude='*.tfstate*' \
            --exclude='.git' .

          # Push to GHCR
          oras push "ghcr.io/${ORG}/module-${MODULE_NAME}:${VERSION}" \
            --config /dev/null:application/vnd.opentofu.module.config.v1+json \
            "${MODULE_NAME}-${VERSION}.tgz:application/vnd.opentofu.module.v1.tar+gzip"

          # Also tag as latest
          oras tag "ghcr.io" \
            "${ORG}/module-${MODULE_NAME}:${VERSION}" \
            "${ORG}/module-${MODULE_NAME}:latest"
```

## Inspecting Pushed Modules

```bash
# List available module versions in OCI
oras repo tags registry.internal.company.com/mycompany/module-vpc

# Inspect the manifest
oras manifest fetch registry.internal.company.com/mycompany/module-vpc:1.0.0

# Pull and inspect module contents
mkdir -p /tmp/module-inspect
oras pull registry.internal.company.com/mycompany/module-vpc:1.0.0 \
  --output /tmp/module-inspect/
tar -tzf /tmp/module-inspect/*.tgz
```

## Version Inventory Script

```bash
#!/bin/bash
# list-oci-modules.sh - Show all modules and versions in OCI registry

REGISTRY="registry.internal.company.com"
NAMESPACE="mycompany"

echo "OpenTofu Modules in OCI Registry"
echo "Registry: $REGISTRY"
echo "================================="

# List all module repositories
oras repo ls "${REGISTRY}/${NAMESPACE}" --prefix module- | while read -r repo; do
  MODULE=$(echo "$repo" | sed 's/module-//')
  TAGS=$(oras repo tags "${REGISTRY}/${NAMESPACE}/${repo}" 2>/dev/null | tr '\n' ', ' | sed 's/,$//')
  echo "  ${MODULE}: ${TAGS}"
done
```

## Conclusion

Pushing modules to OCI registries uses `oras push` with the `application/vnd.opentofu.module.v1.tar+gzip` content type to package module directories as `.tgz` archives. Use semantic version tags (major, minor, patch, and latest) to give consumers flexibility in version constraints. The GitHub Actions pipeline shows the full automated workflow: validate → package → push — triggered on version tags. Once pushed, modules are referenced in configurations using the `oci::` source prefix.
