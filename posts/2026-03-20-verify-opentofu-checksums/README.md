# How to Verify Your OpenTofu Installation with Checksums

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Security, Checksums, Verification, Infrastructure as Code, DevOps

Description: A guide to verifying OpenTofu binary integrity using SHA256 checksums and GPG signature verification.

## Introduction

Verifying the integrity of downloaded binaries is a critical security practice. OpenTofu provides SHA256 checksums and GPG signatures for all release artifacts. This guide covers how to verify your OpenTofu installation to ensure it hasn't been tampered with.

## Why Verification Matters

- Ensures the binary was not corrupted during download
- Protects against supply chain attacks
- Confirms the binary was built and signed by the OpenTofu team
- Required for compliance in security-conscious environments

## Step 1: Download the Release Files

```bash
TOFU_VERSION="1.9.0"
ARCH="linux_amd64"

# Download the binary

curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_${ARCH}.zip"

# Download the checksums file
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_SHA256SUMS"

# Download the checksums signature file
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_SHA256SUMS.sig"

# Download the signing key
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_SHA256SUMS.pem"
```

## Step 2: Verify SHA256 Checksum

```bash
# Verify the checksum of the downloaded zip
sha256sum -c --ignore-missing "tofu_${TOFU_VERSION}_SHA256SUMS"

# Expected output:
# tofu_1.9.0_linux_amd64.zip: OK

# Or manually check
sha256sum "tofu_${TOFU_VERSION}_${ARCH}.zip"
grep "${ARCH}.zip" "tofu_${TOFU_VERSION}_SHA256SUMS"
# The hash values should match
```

## Step 3: Verify the GPG Signature

```bash
# Import the OpenTofu public key
gpg --import "tofu_${TOFU_VERSION}_SHA256SUMS.pem"

# Or import from keyserver
gpg --keyserver keyserver.ubuntu.com --recv-keys <OPENTOFU_KEY_ID>

# Verify the signature
gpg --verify "tofu_${TOFU_VERSION}_SHA256SUMS.sig" "tofu_${TOFU_VERSION}_SHA256SUMS"

# Expected output:
# gpg: Signature made...
# gpg: Good signature from "OpenTofu..."
```

## Step 4: Verify Using cosign (Advanced)

```bash
# Install cosign
curl -LO "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
sudo mv cosign-linux-amd64 /usr/local/bin/cosign
sudo chmod +x /usr/local/bin/cosign

# Verify using sigstore
cosign verify-blob \
  --certificate "tofu_${TOFU_VERSION}_SHA256SUMS.pem" \
  --signature "tofu_${TOFU_VERSION}_SHA256SUMS.sig" \
  --certificate-identity "https://github.com/opentofu/opentofu/.github/workflows/release.yml@refs/tags/v${TOFU_VERSION}" \
  --certificate-oidc-issuer "https://token.actions.githubusercontent.com" \
  "tofu_${TOFU_VERSION}_SHA256SUMS"
```

## Scripted Verification

Automate the entire verification process:

```bash
#!/bin/bash
# verify-tofu.sh - Download and verify OpenTofu

set -euo pipefail

TOFU_VERSION="${1:-1.9.0}"
ARCH="linux_amd64"
BASE_URL="https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}"

echo "Downloading OpenTofu v${TOFU_VERSION}..."

# Download files
for file in \
  "tofu_${TOFU_VERSION}_${ARCH}.zip" \
  "tofu_${TOFU_VERSION}_SHA256SUMS" \
  "tofu_${TOFU_VERSION}_SHA256SUMS.sig" \
  "tofu_${TOFU_VERSION}_SHA256SUMS.pem"; do
  curl -fsSL -O "${BASE_URL}/${file}"
done

echo "Verifying SHA256 checksum..."
sha256sum -c --ignore-missing "tofu_${TOFU_VERSION}_SHA256SUMS"
echo "Checksum verification: PASSED"

echo "Verifying GPG signature..."
gpg --import "tofu_${TOFU_VERSION}_SHA256SUMS.pem" 2>/dev/null
gpg --verify "tofu_${TOFU_VERSION}_SHA256SUMS.sig" "tofu_${TOFU_VERSION}_SHA256SUMS"
echo "GPG signature verification: PASSED"

echo "Extracting binary..."
unzip "tofu_${TOFU_VERSION}_${ARCH}.zip"
echo "Installation ready. Binary: ./tofu"
```

```bash
# Run the verification script
chmod +x verify-tofu.sh
./verify-tofu.sh 1.9.0
```

## Verifying on macOS

```bash
# On macOS, use shasum instead of sha256sum
shasum -a 256 -c --ignore-missing "tofu_${TOFU_VERSION}_SHA256SUMS"
```

## Conclusion

Verifying OpenTofu downloads using checksums and GPG signatures is a fundamental security practice. This is especially important in automated CI/CD pipelines where compromised binaries could affect your entire infrastructure. Always verify before installing, and consider automating verification as part of your deployment scripts.
