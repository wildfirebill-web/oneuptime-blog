# How to Use Docker Hub as OCI Registry for OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Docker Hub, OCI Registry, Provider Distribution, Public Registry

Description: Learn how to use Docker Hub as an OCI registry for distributing OpenTofu providers and modules, leveraging Docker Hub's public and private repository infrastructure.

## Introduction

Docker Hub is OCI-compliant and can store OCI artifacts alongside container images. While ECR, ACR, or GCR are typically preferred for enterprise use, Docker Hub is useful for open-source OpenTofu providers and public modules that need to be freely accessible without cloud provider accounts. Docker Hub's free tier allows public repositories with unlimited pulls.

## Docker Hub Repository Setup

```bash
# Login to Docker Hub

docker login docker.io \
  --username "$DOCKERHUB_USERNAME" \
  --password "$DOCKERHUB_TOKEN"

# Create repositories via Docker Hub UI or CLI
# (Repositories are auto-created on first push)

# For organizations, use organization namespaces:
# docker.io/mycompany/opentofu-provider-myprovider
```

## Pushing a Provider to Docker Hub

```bash
#!/bin/bash
# push-provider-dockerhub.sh

set -euo pipefail

DOCKERHUB_USER="${DOCKERHUB_USERNAME}"
PROVIDER_NAME="myprovider"
VERSION="1.0.0"
REGISTRY="docker.io"

# Login
echo "$DOCKERHUB_TOKEN" | docker login "$REGISTRY" \
  -u "$DOCKERHUB_USER" --password-stdin

# Build provider
mkdir -p dist/
for OS in linux darwin windows; do
  for ARCH in amd64 arm64; do
    GOOS=$OS GOARCH=$ARCH go build \
      -o "dist/terraform-provider-${PROVIDER_NAME}_${VERSION}_${OS}_${ARCH}"
    zip "dist/terraform-provider-${PROVIDER_NAME}_${VERSION}_${OS}_${ARCH}.zip" \
      "dist/terraform-provider-${PROVIDER_NAME}_${VERSION}_${OS}_${ARCH}"
  done
done

cd dist/
sha256sum *.zip > "terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS"
gpg --detach-sign "terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS"

# Push to Docker Hub
REPO="${REGISTRY}/${DOCKERHUB_USER}/opentofu-provider-${PROVIDER_NAME}"

oras push "${REPO}:${VERSION}" \
  --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
  "terraform-provider-${PROVIDER_NAME}_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64" \
  "terraform-provider-${PROVIDER_NAME}_${VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64" \
  "terraform-provider-${PROVIDER_NAME}_${VERSION}_darwin_arm64.zip:application/vnd.opentofu.provider.v1.darwin.arm64" \
  "terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums" \
  "terraform-provider-${PROVIDER_NAME}_${VERSION}_SHA256SUMS.sig:application/vnd.opentofu.provider.v1.shasums.sig"

echo "Pushed: ${REPO}:${VERSION}"
```

## Pushing Modules to Docker Hub

```bash
#!/bin/bash
# push-module-dockerhub.sh

MODULE_DIR="${1:?Usage: $0 <module-dir> <version>}"
VERSION="${2:?}"
DOCKERHUB_USER="${DOCKERHUB_USERNAME}"
MODULE_NAME=$(basename "$MODULE_DIR")

echo "$DOCKERHUB_TOKEN" | docker login docker.io -u "$DOCKERHUB_USER" --password-stdin

tar -czf "${MODULE_NAME}-${VERSION}.tgz" \
  --exclude='.terraform' \
  --exclude='*.tfstate*' \
  --exclude='.git' \
  -C "$MODULE_DIR" .

REPO="docker.io/${DOCKERHUB_USER}/opentofu-module-${MODULE_NAME}"

oras push "${REPO}:${VERSION}" \
  --config /dev/null:application/vnd.opentofu.module.config.v1+json \
  "${MODULE_NAME}-${VERSION}.tgz:application/vnd.opentofu.module.v1.tar+gzip"

# Tag as latest
oras tag "docker.io" \
  "${DOCKERHUB_USER}/opentofu-module-${MODULE_NAME}:${VERSION}" \
  "${DOCKERHUB_USER}/opentofu-module-${MODULE_NAME}:latest"

rm "${MODULE_NAME}-${VERSION}.tgz"
echo "Module available at: ${REPO}:${VERSION}"
```

## Configuring OpenTofu to Use Docker Hub

```hcl
# ~/.terraform.rc - for public Docker Hub repositories (no auth needed)

provider_installation {
  oci_mirror {
    url     = "oci://docker.io/mycompany"
    include = ["registry.opentofu.org/mycompany/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/mycompany/*"]
  }
}
```

```hcl
# For private Docker Hub repositories, add credentials
credentials "docker.io" {
  # Docker Hub uses token-based auth
  # Set via DOCKERHUB_TOKEN environment variable
  # or docker login configures ~/.docker/config.json
}
```

## Using Public Modules from Docker Hub

```hcl
# Reference a public module from Docker Hub
module "vpc" {
  source = "oci://docker.io/mycompany/opentofu-module-vpc:1.0.0"

  name = "production"
  cidr = "10.0.0.0/16"
}

# Public repositories don't require authentication
# Anyone can pull without credentials
```

## GitHub Actions for Publishing to Docker Hub

```yaml
# .github/workflows/publish-to-dockerhub.yml
name: Publish to Docker Hub

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'

      - name: Extract version
        id: version
        run: echo "version=${GITHUB_REF_NAME#v}" >> $GITHUB_OUTPUT

      - name: Install oras
        run: |
          curl -LO https://github.com/oras-project/oras/releases/download/v1.1.0/oras_1.1.0_linux_amd64.tar.gz
          tar -xzf oras_1.1.0_linux_amd64.tar.gz
          sudo mv oras /usr/local/bin/

      - name: Login to Docker Hub
        run: |
          echo "${{ secrets.DOCKERHUB_TOKEN }}" | \
            docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin

      - name: Build and push provider
        run: |
          VERSION="${{ steps.version.outputs.version }}"
          make package VERSION="$VERSION"
          cd dist/

          oras push "docker.io/${{ secrets.DOCKERHUB_USERNAME }}/opentofu-provider-myprovider:${VERSION}" \
            --config /dev/null:application/vnd.opentofu.provider.config.v1+json \
            "terraform-provider-myprovider_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64" \
            "terraform-provider-myprovider_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums"

      - name: Update Docker Hub description
        uses: peter-evans/dockerhub-description@v4
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
          repository: ${{ secrets.DOCKERHUB_USERNAME }}/opentofu-provider-myprovider
          readme-filepath: ./README.md
```

## Docker Hub Rate Limits

```bash
# Docker Hub rate limits anonymous pulls: 100 pulls/6 hours
# Authenticated pulls: 200 pulls/6 hours
# Docker Hub Pro/Team: unlimited pulls

# Always authenticate in CI/CD to avoid rate limits
docker login docker.io \
  --username "$DOCKERHUB_USERNAME" \
  --password "$DOCKERHUB_TOKEN"

# Check your current rate limit status
TOKEN=$(curl -s "https://auth.docker.io/token?service=registry.docker.io&scope=repository:ratelimitpreview/test:pull" | jq -r .token)
curl -s --head -H "Authorization: Bearer $TOKEN" https://registry-1.docker.io/v2/ratelimitpreview/test/manifests/latest | grep -i ratelimit
```

## Conclusion

Docker Hub works as an OCI registry for OpenTofu providers and modules, with the main advantage being free public repositories accessible without cloud provider accounts. Use Docker Hub for open-source providers that anyone should be able to use without authentication. For enterprise use with private repositories, the Docker Hub Pro plan removes rate limits. In CI/CD pipelines, always authenticate even for public repositories to avoid anonymous pull rate limits.
