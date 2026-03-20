# How to Install OpenTofu on Fedora

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Fedora, Installation, Infrastructure as Code, DevOps

Description: A step-by-step guide to installing OpenTofu on Fedora Linux using the official RPM repository and other methods.

## Introduction

Fedora is a community-driven Linux distribution sponsored by Red Hat, known for incorporating cutting-edge software. This guide walks you through installing OpenTofu on Fedora using the official RPM repository.

## Prerequisites

- Fedora 38 or later
- `sudo` privileges
- Internet access

## Method 1: Install via the Official YUM/DNF Repository

### Step 1: Add the OpenTofu Repository

```bash
# Add the OpenTofu repository using dnf config-manager

sudo dnf config-manager --add-repo https://packages.opentofu.org/opentofu/tofu/config_file.repo?type=rpm-md

# Or manually create the repo file
cat <<EOF | sudo tee /etc/yum.repos.d/opentofu.repo
[opentofu]
name=opentofu
baseurl=https://packages.opentofu.org/opentofu/tofu/rpm_any/rpm_any/\$basearch
repo_gpgcheck=1
gpgcheck=1
enabled=1
gpgkey=https://get.opentofu.org/opentofu.gpg
       https://packages.opentofu.org/opentofu/tofu/gpgkey
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
EOF
```

### Step 2: Install OpenTofu

```bash
# Install OpenTofu using dnf
sudo dnf install -y tofu
```

## Method 2: Install from RPM Package

Download and install the RPM package directly:

```bash
TOFU_VERSION="1.9.0"

# Download the RPM package
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.rpm"

# Install using rpm
sudo rpm -i "tofu_${TOFU_VERSION}_linux_amd64.rpm"

# Or using dnf for better dependency management
sudo dnf install "tofu_${TOFU_VERSION}_linux_amd64.rpm"
```

## Method 3: Install from Binary

```bash
TOFU_VERSION="1.9.0"

# Download and extract
curl -LO "https://github.com/opentofu/opentofu/releases/download/v${TOFU_VERSION}/tofu_${TOFU_VERSION}_linux_amd64.zip"
unzip "tofu_${TOFU_VERSION}_linux_amd64.zip"

# Install to system path
sudo mv tofu /usr/local/bin/
sudo chmod +x /usr/local/bin/tofu
```

## Verifying the Installation

```bash
# Check installed version
tofu version

# Output:
# OpenTofu v1.9.0
# on linux_amd64
```

## Setting Up Shell Autocompletion

```bash
# For bash
tofu -install-autocomplete
source ~/.bashrc

# For zsh
tofu -install-autocomplete
source ~/.zshrc
```

## Quick Start Example

```hcl
# main.tf - Testing OpenTofu on Fedora
terraform {
  required_version = ">= 1.6"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "hello" {
  content  = "Hello from OpenTofu on Fedora!"
  filename = "${path.module}/hello.txt"
}

output "file_path" {
  value = local_file.hello.filename
}
```

```bash
# Initialize and apply
tofu init
tofu apply -auto-approve

# Verify the file was created
cat hello.txt
```

## Updating OpenTofu

```bash
# Update using dnf
sudo dnf update tofu

# Verify the new version
tofu version
```

## Removing OpenTofu

```bash
# Remove using dnf
sudo dnf remove tofu

# Also remove the repository
sudo rm /etc/yum.repos.d/opentofu.repo
```

## Conclusion

Installing OpenTofu on Fedora is simple using the official DNF repository. The package manager approach makes it easy to keep OpenTofu updated and manage its lifecycle. You are now ready to use OpenTofu to define and provision infrastructure on Fedora Linux.
