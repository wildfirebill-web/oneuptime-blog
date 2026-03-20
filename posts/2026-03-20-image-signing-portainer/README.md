# How to Set Up Image Signing and Verification in Portainer

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Docker, Security, Image Signing, Cosign, Supply Chain Security

Description: Sign Docker images with Cosign and configure Portainer to only run verified, signed images as part of a supply chain security strategy.

## Introduction

Container image signing ensures that the images you deploy are exactly what was built and hasn't been tampered with in transit. This is supply chain security for containers. Cosign (from the Sigstore project) is the modern standard for signing container images, providing keyless signing via OIDC or traditional key-based signing. This guide covers signing images with Cosign and verifying signatures before deploying through Portainer.

## Step 1: Install Cosign

```bash
# Install Cosign on your build machine
curl -O -L https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Verify installation
cosign version

# Generate a signing key pair
cosign generate-key-pair
# Creates: cosign.key (private key), cosign.pub (public key)
# Keep cosign.key secret! cosign.pub is for verification.
```

## Step 2: Sign an Image After Building

```bash
# Build and push your image first
docker build -t registry.example.com/myapp/api:1.0.0 .
docker push registry.example.com/myapp/api:1.0.0

# Sign the image with your private key
cosign sign \
  --key cosign.key \
  registry.example.com/myapp/api:1.0.0

# The signature is stored in the registry as an OCI artifact
# alongside the image itself

# Sign with additional annotations (metadata)
cosign sign \
  --key cosign.key \
  --annotations "git-commit=$(git rev-parse HEAD)" \
  --annotations "build-date=$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
  --annotations "signed-by=ci-pipeline" \
  registry.example.com/myapp/api:1.0.0
```

## Step 3: Verify Image Signatures

```bash
# Verify before pulling and running
cosign verify \
  --key cosign.pub \
  registry.example.com/myapp/api:1.0.0

# Successful verification output:
# Verification for registry.example.com/myapp/api:1.0.0 --
# The following checks were performed on each of these signatures:
#   - The cosign claims were validated
#   - The signatures were verified against the specified public key

# Verify and check annotations
cosign verify \
  --key cosign.pub \
  --annotations "signed-by=ci-pipeline" \
  registry.example.com/myapp/api:1.0.0

# Get signing metadata
cosign triangulate registry.example.com/myapp/api:1.0.0
# Shows the signature reference in the registry
```

## Step 4: Keyless Signing with GitHub Actions

```yaml
# .github/workflows/build-sign.yml
name: Build and Sign

on:
  push:
    branches: [main]

permissions:
  contents: read
  packages: write
  id-token: write  # Required for keyless signing

jobs:
  build-and-sign:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Cosign
        uses: sigstore/cosign-installer@v3

      - name: Log in to Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and Push
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}

      - name: Sign Image (Keyless - uses GitHub OIDC)
        run: |
          cosign sign \
            --yes \
            ghcr.io/${{ github.repository }}:${{ github.sha }}
          # Keyless: signature stored in Rekor public ledger
```

## Step 5: Enforce Signature Verification Before Deployment

```bash
#!/bin/bash
# verify-and-deploy.sh - Verify signature before triggering Portainer deployment

IMAGE=$1
COSIGN_PUBLIC_KEY=/etc/cosign/cosign.pub
PORTAINER_WEBHOOK="https://portainer.example.com/api/webhooks/YOUR_WEBHOOK_ID"

echo "Verifying signature for: $IMAGE"

cosign verify \
  --key "$COSIGN_PUBLIC_KEY" \
  "$IMAGE" > /dev/null 2>&1

if [ $? -ne 0 ]; then
  echo "BLOCKED: Image signature verification failed for $IMAGE"
  echo "This image has not been signed by the CI pipeline."
  exit 1
fi

echo "Signature verified. Proceeding with deployment."

# Update compose stack image reference
sed -i "s|image: myapp/api:.*|image: $IMAGE|" /opt/stacks/myapp/docker-compose.yml

# Trigger Portainer deployment
curl -s -X POST "$PORTAINER_WEBHOOK"
echo "Deployment triggered."
```

## Step 6: Container Signature Policy with Connaisseur

```yaml
# Connaisseur enforces image verification as a Kubernetes admission webhook
# For Docker, use a verification wrapper script

# docker-compose.yml with wrapper image that verifies before starting
version: "3.8"

services:
  api:
    image: registry.example.com/myapp/api:1.0.0
    # Use a pre-start hook to verify the image signature
    entrypoint: ["/bin/sh", "-c"]
    command:
      - |
        echo "Verifying image signature..."
        cosign verify \
          --key /etc/cosign/cosign.pub \
          registry.example.com/myapp/api:1.0.0 || exit 1
        echo "Signature valid. Starting application..."
        exec node server.js
    volumes:
      - ./cosign.pub:/etc/cosign/cosign.pub:ro
```

## Step 7: Verify All Running Container Images

```bash
# Audit script - verify signatures on all running containers
#!/bin/bash

COSIGN_KEY=/etc/cosign/cosign.pub
FAILURES=0

echo "Auditing running containers for valid signatures..."

docker ps --format "{{.Image}}" | sort -u | while read image; do
  echo -n "Checking $image... "
  if cosign verify --key "$COSIGN_KEY" "$image" > /dev/null 2>&1; then
    echo "VALID"
  else
    echo "UNSIGNED or INVALID"
    FAILURES=$((FAILURES + 1))
  fi
done

if [ $FAILURES -gt 0 ]; then
  echo "WARNING: $FAILURES images failed signature verification"
  exit 1
fi
echo "All images verified successfully."
```

## Conclusion

Image signing creates a cryptographic chain of custody from build to deployment. Every signed image proves it came from your CI/CD pipeline and hasn't been modified. Cosign's keyless mode using OIDC tokens (GitHub Actions, GitLab CI) eliminates key management overhead while maintaining cryptographic guarantees. Combining signature verification with Portainer webhook deployments creates a secure deployment gate — only images that passed CI and were signed can reach production.
