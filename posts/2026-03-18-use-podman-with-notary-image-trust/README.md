# How to Use Podman with Notary for Image Trust

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Notary, Image Trust, Content Trust, Container Security

Description: Learn how to use Notary with Podman to establish image trust through content signing and verification, ensuring only trusted images are deployed.

---

> Notary provides a trust framework for your Podman container images, ensuring that every image pulled from a registry has been signed by a trusted publisher and has not been modified in transit.

Container image trust is a fundamental security requirement for production deployments. Notary, based on The Update Framework (TUF), provides a robust system for signing and verifying container image content. It protects against several attack vectors including compromised registry servers, man-in-the-middle attacks, and replay attacks. Integrating Notary with Podman ensures that your container runtime only accepts images that have been explicitly signed and verified.

---

## Understanding Notary and TUF

Notary implements The Update Framework, which uses a hierarchy of cryptographic keys to establish trust. The key hierarchy includes:

- Root key: The master key that signs all other keys (kept offline)
- Targets key: Signs the actual container image metadata
- Snapshot key: Signs a collection of all target metadata
- Timestamp key: Provides freshness guarantees to prevent replay attacks

This separation of concerns means that compromising a single key does not compromise the entire trust chain.

## Setting Up a Notary Server

Deploy Notary alongside your container registry:

```yaml
# notary-stack.yml
version: "3"
services:
  notary-server:
    image: notary:server
    restart: always
    ports:
      - "4443:4443"
    volumes:
      - ./notary/server-config.json:/etc/notary/server-config.json:ro
      - ./notary/fixtures:/etc/notary/fixtures:ro
    depends_on:
      - notary-db

  notary-signer:
    image: notary:signer
    restart: always
    volumes:
      - ./notary/signer-config.json:/etc/notary/signer-config.json:ro
      - ./notary/fixtures:/etc/notary/fixtures:ro
    depends_on:
      - notary-db

  notary-db:
    image: mariadb:10.11
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: notarydbpass
      MYSQL_DATABASE: notaryserver
    volumes:
      - notary-db-data:/var/lib/mysql

  registry:
    image: registry:2
    restart: always
    ports:
      - "5000:5000"
    volumes:
      - registry-data:/var/lib/registry

volumes:
  notary-db-data:
  registry-data:
```

Notary server configuration:

```json
{
    "server": {
        "http_addr": ":4443",
        "tls_key_file": "/etc/notary/fixtures/notary-server.key",
        "tls_cert_file": "/etc/notary/fixtures/notary-server.crt"
    },
    "trust_service": {
        "type": "remote",
        "hostname": "notary-signer",
        "port": "7899",
        "tls_ca_file": "/etc/notary/fixtures/root-ca.crt",
        "key_algorithm": "ecdsa"
    },
    "storage": {
        "backend": "mysql",
        "db_url": "root:notarydbpass@tcp(notary-db:3306)/notaryserver"
    }
}
```

## Initializing Trust for a Repository

Initialize content trust for a container image repository:

```bash
# Initialize trust for a repository using the notary CLI
# Note: Podman does not support the DOCKER_CONTENT_TRUST environment variable.
# Use the notary CLI directly and Podman's native policy.json for trust enforcement.
notary -s https://notary.example.com:4443 init registry.example.com/myapp

# This generates the root, targets, snapshot, and timestamp keys
# Keep the root key secure and offline
```

## Signing Images

Sign and push a trusted image:

```bash
# Build the image
podman build -t registry.example.com/myapp:v1.0.0 .

# Push and sign the image using Podman's native GPG signing
podman push --sign-by image-signing@example.com \
  registry.example.com/myapp:v1.0.0

# Or sign using notary directly
notary addhash \
  registry.example.com/myapp \
  v1.0.0 \
  <sha256-digest> \
  --publish
```

## Configuring Podman for Trust Verification

Configure Podman to verify image signatures:

```yaml
# /etc/containers/registries.d/default.yaml
docker:
  registry.example.com:
    sigstore: https://notary.example.com/signatures
    use-sigstore-attachments: true
```

```json
// /etc/containers/policy.json
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports": {
        "docker": {
            "registry.example.com": [
                {
                    "type": "signedBy",
                    "keyType": "GPGKeys",
                    "keyPath": "/etc/pki/containers/trusted-key.gpg"
                }
            ],
            "docker.io": [
                {
                    "type": "insecureAcceptAnything"
                }
            ]
        }
    }
}
```

With this configuration, Podman rejects any unsigned image from `registry.example.com`:

```bash
# This will succeed if the image is signed
podman pull registry.example.com/myapp:v1.0.0

# This will fail if the image is not signed
podman pull registry.example.com/untrusted:latest
# Error: image trust policy rejection
```

## GPG-Based Image Signing with Podman

Podman supports GPG-based image signing natively:

```bash
# Generate a GPG key for image signing
gpg --quick-generate-key image-signing@example.com

# Configure signing for a registry
# /etc/containers/registries.d/registry.yaml
docker:
  registry.example.com:
    sigstore: file:///var/lib/containers/sigstore

# Sign an image with GPG
podman push --sign-by image-signing@example.com \
  registry.example.com/myapp:v1.0.0

# Verify the signature
podman pull registry.example.com/myapp:v1.0.0
# Podman automatically verifies against the policy
```

## Delegation and Team Workflows

Notary supports delegation, allowing different team members to sign different image tags:

```bash
# Add a delegation for the "releases" role
notary delegation add \
  registry.example.com/myapp \
  targets/releases \
  --paths "" \
  --key delegation.crt

# Sign as a delegated signer
notary addhash \
  registry.example.com/myapp \
  v1.0.0 \
  <sha256-digest> \
  --roles targets/releases \
  --publish
```

## Build and Sign Pipeline

A complete CI/CD pipeline with trust:

```bash
#!/bin/bash
# trusted-build.sh

set -euo pipefail

IMAGE="registry.example.com/myapp"
VERSION="$1"
GPG_KEY="image-signing@example.com"

echo "=== Building ${IMAGE}:${VERSION} ==="
podman build -t "${IMAGE}:${VERSION}" .

echo "=== Running security scan ==="
trivy image --exit-code 1 --severity CRITICAL "${IMAGE}:${VERSION}"

echo "=== Pushing and signing ==="
podman push --sign-by "$GPG_KEY" "${IMAGE}:${VERSION}"

echo "=== Verifying trust ==="
# Reset local image to force pull with verification
podman rmi "${IMAGE}:${VERSION}" 2>/dev/null || true
podman pull "${IMAGE}:${VERSION}"

echo "=== Trust chain verified ==="
echo "Image ${IMAGE}:${VERSION} is built, scanned, signed, and verified"
```

## Monitoring Trust Status

Check the trust status of images in your registry:

```bash
#!/bin/bash
# audit-trust.sh

REGISTRY="registry.example.com"

echo "Image Trust Audit - $(date)"
echo "=========================="

# List all repositories
REPOS=$(curl -s "https://${REGISTRY}/v2/_catalog" | jq -r '.repositories[]')

for repo in $REPOS; do
    TAGS=$(curl -s "https://${REGISTRY}/v2/${repo}/tags/list" | jq -r '.tags[]?' 2>/dev/null)

    for tag in $TAGS; do
        IMAGE="${REGISTRY}/${repo}:${tag}"
        if podman pull --quiet "$IMAGE" > /dev/null 2>&1; then
            echo "TRUSTED:   $IMAGE"
        else
            echo "UNTRUSTED: $IMAGE"
        fi
    done
done
```

## Key Rotation

Rotate signing keys periodically:

```bash
# Rotate the targets key
notary key rotate registry.example.com/myapp targets

# Rotate the snapshot key
notary key rotate registry.example.com/myapp snapshot

# Rotate the timestamp key (usually managed by the server)
notary key rotate registry.example.com/myapp timestamp --server-managed
```

## Emergency Key Revocation

If a signing key is compromised:

```bash
# Remove the compromised delegation
notary delegation remove \
  registry.example.com/myapp \
  targets/releases \
  --key compromised-key-id \
  --publish

# Generate new keys and re-sign all images
notary key rotate registry.example.com/myapp targets --publish
```

## Conclusion

Notary and Podman together provide a robust image trust framework based on The Update Framework. By signing images at build time and verifying signatures at pull time, you create a chain of trust that prevents unauthorized or tampered images from running in your environment. Podman's native support for signature policies means verification can be enforced at the container runtime level, not just as a CI check. Combined with key delegation for team workflows and key rotation for ongoing security, Notary provides enterprise-grade image trust that scales with your organization.
