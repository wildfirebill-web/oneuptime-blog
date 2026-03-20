# How to Use tofu providers mirror for Air-Gapped Environments

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Providers

Description: Learn how to use the tofu providers mirror command to download provider binaries for transfer to air-gapped environments that have no internet access.

## Introduction

The `tofu providers mirror` command downloads all provider binaries required by a configuration into a local directory. The resulting directory structure matches the registry format, making it directly usable as a filesystem mirror. This is the recommended workflow for populating provider mirrors for air-gapped environments.

## Basic Usage

```bash
# Download providers for the current configuration into ./mirror-dir
tofu providers mirror ./mirror-dir
```

The command reads the current configuration's `required_providers` and `.terraform.lock.hcl` to determine which providers to download.

## Downloading for Multiple Platforms

By default, only the current platform's binary is downloaded. Specify all target platforms:

```bash
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  /tmp/provider-mirror
```

## Output Directory Structure

```
/tmp/provider-mirror/
└── registry.opentofu.org/
    └── hashicorp/
        ├── aws/
        │   └── 5.31.0/
        │       ├── terraform-provider-aws_5.31.0_linux_amd64.zip
        │       └── terraform-provider-aws_5.31.0_linux_arm64.zip
        └── kubernetes/
            └── 2.25.0/
                └── terraform-provider-kubernetes_2.25.0_linux_amd64.zip
```

## Complete Air-Gap Workflow

### Step 1: Download on a Connected Machine

```bash
# On the internet-connected machine
mkdir -p /tmp/tofu-mirror

# Navigate to your configuration directory
cd /path/to/terraform-config

# Initialize to get the lock file
tofu init

# Mirror all providers for target platforms
tofu providers mirror \
  -platform=linux_amd64 \
  /tmp/tofu-mirror

# Package the mirror
tar -czf tofu-provider-mirror.tar.gz -C /tmp tofu-mirror
```

### Step 2: Transfer to Air-Gapped Environment

```bash
# Transfer via approved mechanism (USB, secure file transfer, etc.)
scp tofu-provider-mirror.tar.gz airgapped-bastion:/opt/

# Also transfer the lock file and configuration
scp -r /path/to/terraform-config airgapped-bastion:/opt/infra/
```

### Step 3: Configure OpenTofu on the Air-Gapped Machine

```bash
# Extract the mirror
tar -xzf /opt/tofu-provider-mirror.tar.gz -C /opt

# Configure OpenTofu to use only the local mirror
cat > ~/.tofurc << 'EOF'
provider_installation {
  filesystem_mirror {
    path    = "/opt/tofu-mirror"
    include = ["registry.opentofu.org/*/*"]
  }
}
EOF
```

### Step 4: Initialize Without Internet

```bash
cd /opt/infra
tofu init  # Uses local mirror — no internet required

# Output:
# Initializing provider plugins...
# - Installing hashicorp/aws v5.31.0... (from local mirror)
```

## Updating the Mirror

When provider versions change:

```bash
# On the connected machine: update lock file and re-mirror
tofu init -upgrade
tofu providers mirror -platform=linux_amd64 /tmp/tofu-mirror-updated
tar -czf tofu-provider-mirror-v2.tar.gz -C /tmp tofu-mirror-updated
# Transfer and replace the old mirror on the air-gapped machine
```

## Conclusion

`tofu providers mirror` is the essential tool for populating offline provider mirrors. Download providers on a connected machine, package the mirror directory, transfer it to the air-gapped environment, and configure `.tofurc` to use only the local mirror. Repeat the process each time provider versions are updated to keep the air-gapped environment current.
