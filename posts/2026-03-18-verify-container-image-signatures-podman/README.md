# How to Verify Container Image Signatures with Podman

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Podman, Containers, DevOps, Security, Signature Verification, GPG

Description: Learn how to verify container image signatures in Podman to ensure images are authentic and untampered before running them.

---

> Verification is the other half of signing. Without it, signatures are just bytes on disk that nobody checks.

Signing images is only useful if you actually verify those signatures before running containers. Podman can be configured to automatically reject images that fail signature verification, ensuring that only authenticated images enter your environment. This guide covers setting up and testing signature verification.

---

## How Signature Verification Works

When Podman pulls an image, it checks the trust policy in `policy.json`. If the policy requires a signature, Podman downloads the signature from the configured sigstore location, verifies it against the specified public key, and only completes the pull if the signature is valid.

## Setting Up Public Keys for Verification

First, install the public keys of your trusted image signers.

```bash
# Create the directory for trusted public keys
sudo mkdir -p /etc/pki/containers

# Import a public key from a file
sudo cp container-signer-public.gpg /etc/pki/containers/

# Or import from a GPG keyring
gpg --export container-signer@example.com | sudo tee /etc/pki/containers/container-signer.gpg > /dev/null
```

```bash
# Verify the key is in place
ls -la /etc/pki/containers/
```

## Configuring the Verification Policy

Set up `policy.json` to require signature verification for specific registries.

```bash
# Back up current policy
sudo cp /etc/containers/policy.json /etc/containers/policy.json.bak

# Configure signature verification
sudo tee /etc/containers/policy.json > /dev/null << 'EOF'
{
  "default": [
    {
      "type": "insecureAcceptAnything"
    }
  ],
  "transports": {
    "docker": {
      "registry.example.com": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/containers/container-signer.gpg"
        }
      ],
      "localhost:5000": [
        {
          "type": "signedBy",
          "keyType": "GPGKeys",
          "keyPath": "/etc/pki/containers/container-signer.gpg"
        }
      ]
    }
  }
}
EOF
```

## Configuring Signature Lookup Locations

Tell Podman where to find signatures for verification.

```bash
# Configure signature sources for the registry
sudo tee /etc/containers/registries.d/myregistry.yaml > /dev/null << 'EOF'
docker:
  registry.example.com:
    sigstore: https://sigstore.example.com/signatures
  localhost:5000:
    sigstore: file:///var/lib/containers/sigstore
EOF
```

## Testing Signature Verification

```bash
# Start a local registry for testing
podman run --rm -d -p 5000:5000 --name test-registry docker.io/library/registry:2

# Push a signed image
podman tag docker.io/library/alpine:latest localhost:5000/alpine:signed
podman push --tls-verify=false \
  --sign-by container-signer@example.com \
  localhost:5000/alpine:signed
```

```bash
# Remove the local copy
podman rmi localhost:5000/alpine:signed

# Pull with signature verification enabled
podman pull --tls-verify=false localhost:5000/alpine:signed
# This should succeed if the signature is valid
```

## Testing Verification Failure

```bash
# Push an unsigned image to the same registry
podman tag docker.io/library/alpine:latest localhost:5000/alpine:unsigned
podman push --tls-verify=false localhost:5000/alpine:unsigned

# Remove local copy
podman rmi localhost:5000/alpine:unsigned

# Attempt to pull the unsigned image (should fail with signedBy policy)
podman pull --tls-verify=false localhost:5000/alpine:unsigned 2>&1 || \
  echo "Verification failed: image is not signed"
```

## Using skopeo for Signature Inspection

```bash
# Install skopeo if needed
# sudo dnf install -y skopeo

# Inspect the signatures of an image
skopeo inspect --raw docker://localhost:5000/alpine:signed --tls-verify=false

# Standalone signature verification with skopeo
skopeo standalone-verify \
  manifest.json \
  localhost:5000/alpine:signed \
  /etc/pki/containers/container-signer.gpg \
  signature-file 2>/dev/null || echo "Use skopeo for advanced verification"
```

## Verifying Red Hat Container Images

Red Hat images come pre-signed and can be verified on RHEL/Fedora systems.

```bash
# Red Hat provides pre-configured signature verification
# Check if the Red Hat signing key is installed
ls /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release 2>/dev/null && \
  echo "Red Hat signing key found"

# View Red Hat registry trust configuration
cat /etc/containers/registries.d/default.yaml 2>/dev/null

# Pull a verified Red Hat image
podman pull registry.access.redhat.com/ubi9/ubi-minimal:latest
```

## Creating a Verification Test Script

```bash
#!/bin/bash
# verify-image.sh - Verify an image's signature before running it

IMAGE=$1

if [ -z "$IMAGE" ]; then
  echo "Usage: $0 <image-reference>"
  exit 1
fi

echo "Verifying image: $IMAGE"

# Attempt to pull with policy enforcement
if podman pull "$IMAGE" 2>/dev/null; then
  echo "[PASS] Image pulled and verified successfully"

  # Show the image digest for auditing
  digest=$(podman inspect "$IMAGE" --format '{{.Digest}}')
  echo "Image digest: $digest"
else
  echo "[FAIL] Image verification failed - image may be unsigned or tampered"
  exit 1
fi
```

```bash
# Make the script executable and test it
chmod +x verify-image.sh
./verify-image.sh docker.io/library/alpine:latest
```

## Debugging Verification Issues

```bash
# Pull with debug logging to see signature verification details
podman pull --log-level debug localhost:5000/alpine:signed --tls-verify=false 2>&1 | \
  grep -i "signature\|policy\|trust\|verify"
```

## Restoring Default Policy

```bash
sudo cp /etc/containers/policy.json.bak /etc/containers/policy.json 2>/dev/null
```

## Cleanup

```bash
podman stop test-registry 2>/dev/null
podman rm test-registry 2>/dev/null
rm -f verify-image.sh
```

## Summary

Verifying container image signatures with Podman closes the trust loop between image publishers and consumers. By configuring trust policies that require signatures, setting up public key paths, and specifying signature lookup locations, you create an automated verification gate that prevents unsigned or tampered images from running. Always test your verification setup by confirming that signed images pull successfully and unsigned images are rejected.
