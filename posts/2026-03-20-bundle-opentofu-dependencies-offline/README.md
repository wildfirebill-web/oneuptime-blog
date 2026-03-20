# How to Bundle OpenTofu with All Dependencies for Offline Use

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Bundle, Offline, Air-Gapped, DevOps

Description: Learn how to create a self-contained bundle of OpenTofu with all its providers and modules for deployment in completely offline or air-gapped environments.

## Introduction

A self-contained OpenTofu bundle packages the binary, all required provider plugins, and modules into a single transferable archive. Once transferred to an air-gapped environment, teams can run OpenTofu workflows without any internet access. This guide walks through creating, validating, and deploying such a bundle.

## Bundle Structure

```
opentofu-bundle/
├── install.sh                    # Automated installer
├── binary/
│   ├── opentofu_1.7.0_linux_amd64.zip
│   └── opentofu_1.7.0_SHA256SUMS
├── providers/
│   └── registry.opentofu.org/
│       └── hashicorp/
│           ├── aws/
│           ├── kubernetes/
│           └── helm/
├── modules/
│   ├── terraform-aws-vpc/
│   ├── terraform-aws-eks/
│   └── internal/
└── config/
    └── terraform.rc             # CLI configuration
```

## Creating the Bundle Script

```bash
#!/bin/bash
# create-bundle.sh - Run on an internet-connected machine

set -euo pipefail

TOFU_VERSION="${TOFU_VERSION:-1.7.0}"
BUNDLE_NAME="opentofu-bundle-${TOFU_VERSION}-$(date +%Y%m%d)"
BUNDLE_DIR="/tmp/${BUNDLE_NAME}"
CONFIG_SOURCE="${CONFIG_SOURCE:-./infrastructure}"  # Path to your OpenTofu configs

echo "Creating OpenTofu bundle: $BUNDLE_NAME"

# Create directory structure
mkdir -p "$BUNDLE_DIR"/{binary,providers,modules,config}

# ----------------------------------------
# 1. Download OpenTofu binary
# ----------------------------------------
echo "Downloading OpenTofu ${TOFU_VERSION}..."

for PLATFORM in linux_amd64 linux_arm64 darwin_arm64; do
  curl -Lo "$BUNDLE_DIR/binary/opentofu_${TOFU_VERSION}_${PLATFORM}.zip" \
    "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/opentofu_${TOFU_VERSION}_${PLATFORM}.zip" \
    --fail --silent --show-error
done

curl -Lo "$BUNDLE_DIR/binary/opentofu_${TOFU_VERSION}_SHA256SUMS" \
  "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/opentofu_${TOFU_VERSION}_SHA256SUMS"

# ----------------------------------------
# 2. Mirror providers
# ----------------------------------------
echo "Mirroring providers..."

WORK_DIR=$(mktemp -d)
trap "rm -rf $WORK_DIR" EXIT

# Copy the provider configuration
cp "$CONFIG_SOURCE"/**/*.tf "$WORK_DIR/" 2>/dev/null || \
  cp "$CONFIG_SOURCE"/*.tf "$WORK_DIR/" 2>/dev/null || true

cd "$WORK_DIR"
tofu init -backend=false 2>&1

# Mirror for all platforms
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  "$BUNDLE_DIR/providers/"

# ----------------------------------------
# 3. Bundle modules
# ----------------------------------------
echo "Bundling modules..."

if [ -d "$WORK_DIR/.terraform/modules" ]; then
  cp -r "$WORK_DIR/.terraform/modules/"* "$BUNDLE_DIR/modules/" 2>/dev/null || true
fi

# ----------------------------------------
# 4. Create CLI config
# ----------------------------------------
cat > "$BUNDLE_DIR/config/terraform.rc" << 'RC'
provider_installation {
  filesystem_mirror {
    path    = "/opt/opentofu/providers"
    include = ["registry.opentofu.org/*/*"]
  }
  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
RC

# ----------------------------------------
# 5. Create installer script
# ----------------------------------------
cat > "$BUNDLE_DIR/install.sh" << 'INSTALLER'
#!/bin/bash
set -euo pipefail

BUNDLE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
INSTALL_PREFIX="${INSTALL_PREFIX:-/opt/opentofu}"
TOFU_BIN="${TOFU_BIN:-/usr/local/bin/tofu}"

echo "Installing OpenTofu from bundle..."

# Detect platform
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "$ARCH" in
  x86_64) ARCH="amd64" ;;
  aarch64|arm64) ARCH="arm64" ;;
  *) echo "Unsupported architecture: $ARCH"; exit 1 ;;
esac
PLATFORM="${OS}_${ARCH}"

# Install binary
BINARY_ZIP=$(ls "$BUNDLE_DIR/binary/"*"${PLATFORM}.zip" 2>/dev/null | head -1)
if [ -z "$BINARY_ZIP" ]; then
  echo "No binary found for platform $PLATFORM"
  exit 1
fi

echo "Installing binary for $PLATFORM..."
TMPDIR=$(mktemp -d)
unzip -q "$BINARY_ZIP" -d "$TMPDIR"
sudo install -m 755 "$TMPDIR/tofu" "$TOFU_BIN"
rm -rf "$TMPDIR"

# Install providers
echo "Installing providers..."
sudo mkdir -p "${INSTALL_PREFIX}/providers"
sudo cp -r "$BUNDLE_DIR/providers/"* "${INSTALL_PREFIX}/providers/"

# Install modules
if [ -d "$BUNDLE_DIR/modules" ] && [ -n "$(ls -A "$BUNDLE_DIR/modules")" ]; then
  echo "Installing modules..."
  sudo mkdir -p "${INSTALL_PREFIX}/modules"
  sudo cp -r "$BUNDLE_DIR/modules/"* "${INSTALL_PREFIX}/modules/"
fi

# Install CLI config
sudo mkdir -p /etc/opentofu
sudo cp "$BUNDLE_DIR/config/terraform.rc" /etc/opentofu/terraform.rc

echo ""
echo "Installation complete!"
echo "Run: export TF_CLI_CONFIG_FILE=/etc/opentofu/terraform.rc"
echo "Verify: tofu version"
INSTALLER

chmod +x "$BUNDLE_DIR/install.sh"

# ----------------------------------------
# 6. Create the archive
# ----------------------------------------
echo "Creating archive..."
cd /tmp
tar -czf "${BUNDLE_NAME}.tar.gz" "$BUNDLE_NAME/"

ARCHIVE_SIZE=$(du -sh "/tmp/${BUNDLE_NAME}.tar.gz" | cut -f1)
echo ""
echo "Bundle created: /tmp/${BUNDLE_NAME}.tar.gz (${ARCHIVE_SIZE})"
echo "Transfer to air-gapped environment and run: tar -xzf ${BUNDLE_NAME}.tar.gz && ./${BUNDLE_NAME}/install.sh"
```

## Installing the Bundle

```bash
# On the air-gapped machine:

# Transfer the bundle (USB drive, internal file share, etc.)
# Then extract and install
tar -xzf opentofu-bundle-1.7.0-20240101.tar.gz

cd opentofu-bundle-1.7.0-20240101
sudo ./install.sh

# Configure environment
echo 'export TF_CLI_CONFIG_FILE=/etc/opentofu/terraform.rc' >> ~/.bashrc
source ~/.bashrc

# Verify
tofu version
tofu init  # Should use local mirror
```

## Validating the Bundle

```bash
#!/bin/bash
# validate-bundle.sh - Run after installation

echo "=== OpenTofu Bundle Validation ==="

# Check binary
echo -n "OpenTofu binary: "
tofu version && echo "OK" || echo "FAIL"

# Check CLI config
echo -n "CLI config: "
[ -f "$TF_CLI_CONFIG_FILE" ] && echo "OK ($TF_CLI_CONFIG_FILE)" || echo "FAIL"

# Check providers in mirror
echo "Providers in mirror:"
find /opt/opentofu/providers -name "*.zip" | while read -r zip; do
  provider=$(echo "$zip" | sed 's|.*/registry.opentofu.org/||' | sed 's|/terraform-provider.*||')
  version=$(echo "$zip" | grep -oP '\d+\.\d+\.\d+')
  echo "  - $provider v$version"
done

# Test init without internet
echo ""
echo "Testing tofu init (offline)..."
TMPDIR=$(mktemp -d)
cat > "$TMPDIR/main.tf" << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
EOF

cd "$TMPDIR"
tofu init -backend=false && echo "PASS: Provider installation works offline" || echo "FAIL: Provider installation failed"
rm -rf "$TMPDIR"
```

## Conclusion

A bundled OpenTofu deployment packages the binary, providers (via `tofu providers mirror`), and modules into a single transferable archive. The install script detects the platform, installs the binary, copies providers to the filesystem mirror directory, and writes the CLI config to redirect all provider downloads. After installation, set `TF_CLI_CONFIG_FILE` globally for all users so every `tofu init` uses the local mirror automatically.
