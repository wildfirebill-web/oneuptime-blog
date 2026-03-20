# How to Use GitHub Container Registry as OCI Registry for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, GitHub Container Registry, OCI Registry, GHCR, Provider Distribution

Description: Learn how to use GitHub Container Registry (GHCR) as an OCI registry for distributing OpenTofu providers and modules, with tight integration into GitHub Actions workflows.

## Introduction

GitHub Container Registry (GHCR) is OCI-compliant and tightly integrated with GitHub Actions, making it the natural choice for teams already using GitHub for source control and CI/CD. GHCR supports public and private repositories, fine-grained PAT permissions, and OIDC-based authentication from GitHub Actions without storing long-lived credentials.

## GHCR Authentication

```bash
# Personal access token (classic) - read:packages, write:packages

echo "$GITHUB_TOKEN" | docker login ghcr.io \
  -u "$GITHUB_ACTOR" --password-stdin

# Fine-grained PAT - requires "Read packages" or "Write packages" permission
echo "$FINE_GRAINED_TOKEN" | docker login ghcr.io \
  -u "$GITHUB_ACTOR" --password-stdin

# In GitHub Actions: use GITHUB_TOKEN (automatic)
# No secrets needed - GITHUB_TOKEN has packages permissions by default
```

## GHCR Package Visibility

```bash
# Packages inherit repository visibility by default
# For public packages (accessible without auth):
# 1. Go to GitHub.com → Your profile → Packages
# 2. Select the package → Package settings → Change visibility → Public

# Or set via GitHub API
curl -X PATCH \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Content-Type: application/json" \
  "https://api.github.com/orgs/myorg/packages/container/opentofu-providers%2Fhashicorp-aws/visibility" \
  -d '{"visibility": "public"}'
```

## Pushing Providers to GHCR

```bash
#!/bin/bash
# push-provider-ghcr.sh

set -euo pipefail

GITHUB_ORG="${GITHUB_ORG}"
PROVIDER_NAMESPACE="hashicorp"
PROVIDER_TYPE="aws"
PROVIDER_VERSION="5.20.1"
GHCR_REGISTRY="ghcr.io"

# Login using GITHUB_TOKEN
echo "$GITHUB_TOKEN" | docker login "$GHCR_REGISTRY" \
  -u "$GITHUB_ACTOR" --password-stdin

# Download provider
WORK_DIR=$(mktemp -d)
trap "rm -rf $WORK_DIR" EXIT

cat > "$WORK_DIR/versions.tf" << EOF
terraform {
  required_providers {
    aws = {
      source  = "${PROVIDER_NAMESPACE}/${PROVIDER_TYPE}"
      version = "= ${PROVIDER_VERSION}"
    }
  }
}
EOF

cd "$WORK_DIR"
tofu init -backend=false
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  "$WORK_DIR/mirror/"

cd "$WORK_DIR/mirror/registry.opentofu.org/${PROVIDER_NAMESPACE}/${PROVIDER_TYPE}/"

ARTIFACT="${GHCR_REGISTRY}/${GITHUB_ORG}/opentofu-providers/${PROVIDER_NAMESPACE}-${PROVIDER_TYPE}:${PROVIDER_VERSION}"

oras push "$ARTIFACT" \
  --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64" \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64" \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_darwin_arm64.zip:application/vnd.opentofu.provider.v1.darwin.arm64" \
  "terraform-provider-${PROVIDER_TYPE}_${PROVIDER_VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums"

echo "Pushed: $ARTIFACT"
```

## GitHub Actions: Automated Provider Mirror Updates

```yaml
# .github/workflows/update-provider-mirror.yml
name: Update Provider Mirror

on:
  schedule:
    - cron: '0 3 * * 1'  # Weekly on Monday at 3 AM
  workflow_dispatch:

jobs:
  update-mirror:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read

    steps:
      - uses: actions/checkout@v4

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: '1.7.0'

      - name: Install oras
        run: |
          curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
          tar -xzf oras_1.1.0_linux_amd64.tar.gz
          sudo mv oras /usr/local/bin/

      - name: Login to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            oras login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Mirror providers
        run: |
          # Init to get latest compatible versions
          tofu init -backend=false
          tofu providers mirror -platform=linux_amd64 -platform=linux_arm64 /tmp/mirror/

          # Push each provider
          for PROVIDER_DIR in /tmp/mirror/registry.opentofu.org/*/*/; do
            NAMESPACE=$(basename "$(dirname "$PROVIDER_DIR")")
            TYPE=$(basename "$PROVIDER_DIR")

            for VERSION_DIR in "$PROVIDER_DIR"*/; do
              VERSION=$(basename "$VERSION_DIR")
              ARTIFACT="ghcr.io/${{ github.repository_owner }}/opentofu-providers/${NAMESPACE}-${TYPE}:${VERSION}"

              echo "Pushing $ARTIFACT"
              cd "$VERSION_DIR"
              oras push "$ARTIFACT" \
                --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
                *linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64 \
                *SHA256SUMS:application/vnd.opentofu.provider.v1.shasums
            done
          done
```

## GitHub Actions: Publishing Custom Provider

```yaml
# .github/workflows/publish-provider.yml
name: Publish Provider to GHCR

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      contents: read
      id-token: write  # For OIDC

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Install oras
        run: |
          curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
          tar -xzf oras_1.1.0_linux_amd64.tar.gz && sudo mv oras /usr/local/bin/

      - name: Build provider
        run: make package VERSION=${{ steps.version.outputs.version }}

      - name: Login to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            oras login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Push to GHCR
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          ORG="${{ github.repository_owner }}"
          ARTIFACT="ghcr.io/${ORG}/opentofu-provider-myprovider:${VERSION}"

          cd dist/
          oras push "$ARTIFACT" \
            --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
            terraform-provider-myprovider_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64 \
            terraform-provider-myprovider_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums
```

## Configuring OpenTofu to Use GHCR

```hcl
# ~/.terraform.rc

credentials "ghcr.io" {
  # Uses GITHUB_TOKEN environment variable or docker credential store
  token = "ghp_yourPersonalAccessToken"
}

provider_installation {
  oci_mirror {
    url     = "oci://ghcr.io/myorg/opentofu-providers"
    include = ["registry.opentofu.org/hashicorp/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/hashicorp/*"]
  }
}
```

## Conclusion

GHCR is the most seamless OCI registry for GitHub-based workflows because the `GITHUB_TOKEN` in Actions has automatic `packages: write` permission without additional secrets. The OIDC integration means CI/CD systems authenticate without long-lived credentials. For public open-source providers, setting GHCR package visibility to public allows anyone to pull providers without authentication, making distribution frictionless for the open-source community.
