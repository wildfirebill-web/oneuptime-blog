# How to Create Automated Release Pipelines with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, CI/CD, Release Pipeline, Automation

Description: Learn how to build automated release pipelines with Podman that handle versioning, building, testing, signing, and publishing container images.

---

> An automated release pipeline with Podman takes your code from a git tag to a signed, published container image in minutes with zero manual intervention.

A well-designed release pipeline automates the entire journey from source code to a published, verified container image. Podman handles the container building and pushing, while the pipeline orchestrates versioning, testing, scanning, signing, and publishing. This guide shows you how to build a complete automated release pipeline using Podman.

---

## Release Pipeline Overview

An automated release pipeline typically triggers on a git tag and performs a series of steps to produce a production-ready container image.

```bash
#!/bin/bash
# Automated release pipeline steps:
# 1. Triggered by a git tag (e.g., v1.2.3)
# 2. Validate the version format
# 3. Build the container image with Podman
# 4. Run the full test suite
# 5. Scan for vulnerabilities
# 6. Sign the image
# 7. Push to the production registry
# 8. Generate release notes
# 9. Create a GitHub/GitLab release
```

## Version Extraction and Validation

Parse and validate the version from the git tag.

```bash
#!/bin/bash
# scripts/validate-version.sh
# Extract and validate the semantic version from a git tag

GIT_TAG="${CI_TAG:-$(git describe --tags --exact-match 2>/dev/null)}"

if [ -z "$GIT_TAG" ]; then
  echo "ERROR: No git tag found. Release pipeline requires a tag."
  exit 1
fi

# Strip the 'v' prefix if present
VERSION="${GIT_TAG#v}"

# Validate semantic version format (major.minor.patch)
if ! echo "$VERSION" | grep -qE '^[0-9]+\.[0-9]+\.[0-9]+(-[a-zA-Z0-9.]+)?$'; then
  echo "ERROR: Invalid version format: ${VERSION}"
  echo "Expected: MAJOR.MINOR.PATCH (e.g., 1.2.3 or 1.2.3-rc.1)"
  exit 1
fi

# Parse semver components
MAJOR=$(echo "$VERSION" | cut -d. -f1)
MINOR=$(echo "$VERSION" | cut -d. -f2)
PATCH=$(echo "$VERSION" | cut -d. -f3 | cut -d- -f1)
PRERELEASE=$(echo "$VERSION" | grep -oP '(?<=-).+' || true)

echo "Version: ${VERSION}"
echo "Major: ${MAJOR}, Minor: ${MINOR}, Patch: ${PATCH}"
[ -n "$PRERELEASE" ] && echo "Pre-release: ${PRERELEASE}"

# Export for use in subsequent steps
export VERSION MAJOR MINOR PATCH PRERELEASE
```

## Building the Release Image

Build the image with comprehensive metadata and multiple tags.

```bash
#!/bin/bash
# scripts/build-release.sh
# Build the release container image with proper tags and labels

REGISTRY="docker.io/myorg"
APP="myapp"
VERSION="${VERSION}"
COMMIT_SHA="$(git rev-parse HEAD)"
BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)"

# Generate all the tags this release should have
TAGS=(
  "${REGISTRY}/${APP}:${VERSION}"
  "${REGISTRY}/${APP}:${MAJOR}.${MINOR}"
  "${REGISTRY}/${APP}:${MAJOR}"
)

# Only tag as 'latest' if this is not a pre-release
if [ -z "$PRERELEASE" ]; then
  TAGS+=("${REGISTRY}/${APP}:latest")
fi

# Build tag arguments
TAG_ARGS=""
for TAG in "${TAGS[@]}"; do
  TAG_ARGS="${TAG_ARGS} --tag ${TAG}"
done

# Build the image with OCI labels for traceability
podman build \
  ${TAG_ARGS} \
  --label "org.opencontainers.image.version=${VERSION}" \
  --label "org.opencontainers.image.revision=${COMMIT_SHA}" \
  --label "org.opencontainers.image.created=${BUILD_DATE}" \
  --label "org.opencontainers.image.source=https://github.com/myorg/${APP}" \
  --label "org.opencontainers.image.title=${APP}" \
  .

echo "Built image with tags:"
for TAG in "${TAGS[@]}"; do
  echo "  - ${TAG}"
done
```

## Running the Release Test Suite

Execute comprehensive tests before publishing the release.

```bash
#!/bin/bash
# scripts/test-release.sh
# Run the full test suite against the release image

IMAGE="${REGISTRY}/${APP}:${VERSION}"

echo "=== Running release test suite ==="

# Unit tests
echo "--- Unit tests ---"
podman run --rm "$IMAGE" npm test
echo "Unit tests: PASSED"

# Integration tests with a real database
echo "--- Integration tests ---"
podman network create release-test-net
podman run -d --name release-db --network release-test-net \
  -e POSTGRES_PASSWORD=test -e POSTGRES_DB=test \
  postgres:16-alpine

sleep 5
podman exec release-db pg_isready -U postgres

podman run --rm --network release-test-net \
  -e DATABASE_URL="postgresql://postgres:test@release-db:5432/test" \
  "$IMAGE" npm run test:integration
echo "Integration tests: PASSED"

# Cleanup
podman rm -f release-db
podman network rm release-test-net

# Smoke test: verify the container starts and responds
echo "--- Smoke test ---"
podman run -d --name smoke-test -p 9090:8080 "$IMAGE"
sleep 3

HEALTH=$(curl -s http://localhost:9090/health)
if echo "$HEALTH" | grep -q "ok"; then
  echo "Smoke test: PASSED"
else
  echo "Smoke test: FAILED"
  podman logs smoke-test
  podman rm -f smoke-test
  exit 1
fi

podman rm -f smoke-test
echo "=== All release tests passed ==="
```

## Scanning and Signing

Scan for vulnerabilities and sign the release image.

```bash
#!/bin/bash
# scripts/scan-and-sign.sh
# Scan the release image for vulnerabilities and sign it

IMAGE="${REGISTRY}/${APP}:${VERSION}"

echo "=== Scanning release image ==="

# Save image for scanning
podman save -o /tmp/release-image.tar "$IMAGE"

# Scan with Trivy (fail on critical vulnerabilities)
trivy image --input /tmp/release-image.tar \
  --severity CRITICAL \
  --exit-code 1 \
  --format table

if [ $? -ne 0 ]; then
  echo "ERROR: Critical vulnerabilities found. Cannot release."
  rm -f /tmp/release-image.tar
  exit 1
fi

echo "Scan passed: no critical vulnerabilities"
rm -f /tmp/release-image.tar

echo "=== Signing release image ==="

# Sign with cosign after pushing
podman push "$IMAGE"
cosign sign --key "${COSIGN_KEY}" "$IMAGE"

echo "Image signed successfully: ${IMAGE}"

# Verify the signature
cosign verify --key cosign.pub "$IMAGE"
echo "Signature verified"
```

## GitHub Actions Release Pipeline

A complete GitHub Actions workflow for automated releases.

````yaml
# .github/workflows/release.yml
name: Release Pipeline

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      packages: write
      id-token: write
      contents: write

    steps:
      - uses: actions/checkout@v4

      # Extract version from tag
      - name: Extract version
        run: |
          VERSION=${GITHUB_REF_NAME#v}
          MAJOR=$(echo $VERSION | cut -d. -f1)
          MINOR=$(echo $VERSION | cut -d. -f2)
          echo "VERSION=$VERSION" >> $GITHUB_ENV
          echo "MAJOR=$MAJOR" >> $GITHUB_ENV
          echo "MINOR=$MINOR" >> $GITHUB_ENV

      # Build with Podman
      - name: Build release image
        run: |
          podman build \
            --tag ghcr.io/${{ github.repository }}:${{ env.VERSION }} \
            --tag ghcr.io/${{ github.repository }}:${{ env.MAJOR }}.${{ env.MINOR }} \
            --tag ghcr.io/${{ github.repository }}:${{ env.MAJOR }} \
            --tag ghcr.io/${{ github.repository }}:latest \
            .

      # Run tests
      - name: Run tests
        run: |
          podman run --rm \
            ghcr.io/${{ github.repository }}:${{ env.VERSION }} \
            npm test

      # Push to registry
      - name: Push to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
            podman login ghcr.io -u ${{ github.actor }} --password-stdin
          podman push --all-tags ghcr.io/${{ github.repository }}

      # Install and run cosign for signing
      - uses: sigstore/cosign-installer@v3
      - name: Sign image
        run: |
          cosign sign --yes \
            ghcr.io/${{ github.repository }}:${{ env.VERSION }}

      # Create GitHub Release
      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          generate_release_notes: true
          body: |
            ## Container Image
            ```bash
            podman pull ghcr.io/${{ github.repository }}:${{ env.VERSION }}
            ```
````

## Generating a Changelog

Automatically generate a changelog from git commits between releases.

```bash
#!/bin/bash
# scripts/changelog.sh
# Generate a changelog between the current and previous release tags

CURRENT_TAG="${1:-$(git describe --tags --abbrev=0)}"
PREVIOUS_TAG=$(git describe --tags --abbrev=0 "${CURRENT_TAG}^" 2>/dev/null || echo "")

echo "# Changelog: ${CURRENT_TAG}"
echo ""

if [ -z "$PREVIOUS_TAG" ]; then
  echo "Initial release"
  git log --oneline --format="- %s" "${CURRENT_TAG}"
else
  echo "Changes since ${PREVIOUS_TAG}:"
  echo ""

  # Group commits by type
  echo "## Features"
  git log --oneline --format="- %s" "${PREVIOUS_TAG}..${CURRENT_TAG}" | grep -i "feat\|add" || echo "None"

  echo ""
  echo "## Bug Fixes"
  git log --oneline --format="- %s" "${PREVIOUS_TAG}..${CURRENT_TAG}" | grep -i "fix\|bug" || echo "None"

  echo ""
  echo "## Other Changes"
  git log --oneline --format="- %s" "${PREVIOUS_TAG}..${CURRENT_TAG}" | grep -iv "feat\|add\|fix\|bug" || echo "None"
fi
```

## Summary

An automated release pipeline with Podman streamlines the entire process from git tag to published container image. The pipeline validates the version, builds the image with comprehensive OCI labels and semantic version tags, runs the full test suite, scans for vulnerabilities, signs the image with cosign, and pushes to the production registry. Automating this process eliminates human error, ensures consistency, and makes releases predictable and repeatable. The changelog generation and GitHub/GitLab release creation provide documentation and visibility for every release.
