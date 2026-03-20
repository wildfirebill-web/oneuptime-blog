# How to Install OpenTofu Using Homebrew on macOS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, macOS, Homebrew, Installation, Infrastructure as Code, DevOps

Description: A guide to installing and managing OpenTofu on macOS using the Homebrew package manager.

## Introduction

Homebrew is the most popular package manager for macOS, making it the easiest and most recommended way to install OpenTofu on a Mac. This guide walks you through installing OpenTofu using Homebrew and getting started quickly.

## Prerequisites

- macOS 12 (Monterey) or later
- Homebrew installed (see https://brew.sh)
- Xcode Command Line Tools

## Installing Homebrew (if needed)

```bash
# Install Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Add Homebrew to PATH (for Apple Silicon Macs)
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

## Installing OpenTofu via Homebrew

### Method 1: Using the Official Tap

```bash
# Add the OpenTofu tap
brew tap opentofu/tap

# Install OpenTofu
brew install opentofu
```

### Method 2: Using the Core Formula (if available)

```bash
# OpenTofu may be available in homebrew-core
brew install opentofu
```

## Verifying the Installation

```bash
# Check the installed version
tofu version

# Output:
# OpenTofu v1.9.0
# on darwin_arm64

# Check binary location
which tofu
# /opt/homebrew/bin/tofu (Apple Silicon)
# /usr/local/bin/tofu (Intel)
```

## Setting Up Shell Completion

### For Zsh (default on macOS)

```bash
# Install zsh completions
tofu -install-autocomplete

# Or manually add to .zshrc
echo 'autoload -U +X bashcompinit && bashcompinit' >> ~/.zshrc
echo 'complete -o nospace -C /opt/homebrew/bin/tofu tofu' >> ~/.zshrc
source ~/.zshrc
```

### For Bash

```bash
# If using Bash
brew install bash-completion@2

tofu -install-autocomplete
source ~/.bash_profile
```

## Quick Start on macOS

```hcl
# ~/projects/tofu-test/main.tf
terraform {
  required_version = ">= 1.6"

  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.0"
    }
  }
}

resource "local_file" "greeting" {
  content  = "Hello from OpenTofu on macOS!"
  filename = "${path.module}/greeting.txt"
}

output "file_path" {
  value = local_file.greeting.filename
}
```

```bash
mkdir ~/projects/tofu-test && cd ~/projects/tofu-test
# Create main.tf
tofu init
tofu apply -auto-approve
cat greeting.txt
```

## Managing Multiple Versions with Homebrew

```bash
# Install a specific version using tofuenv (recommended for multiple versions)
brew install tofuenv

# Or use the Homebrew versioned formula
brew install opentofu@1.8

# Switch between versions
brew link --overwrite opentofu@1.8
```

## Updating OpenTofu

```bash
# Update Homebrew and upgrade OpenTofu
brew update && brew upgrade opentofu

# Check new version
tofu version
```

## Uninstalling OpenTofu

```bash
# Remove OpenTofu
brew uninstall opentofu

# Optionally remove the tap
brew untap opentofu/tap

# Clean up
brew cleanup
```

## Tips for macOS Development

```bash
# Create a project directory structure
mkdir -p ~/Projects/infrastructure/{modules,environments}
cd ~/Projects/infrastructure

# Initialize a new project
cat > main.tf <<'EOF'
terraform {
  required_version = ">= 1.6"
}

output "hello" {
  value = "OpenTofu on macOS is working!"
}
EOF

tofu init && tofu apply -auto-approve
```

## Conclusion

Installing OpenTofu via Homebrew on macOS provides the smoothest experience for Mac developers. Homebrew handles dependency management, binary installation, and updates automatically. With OpenTofu installed, you're ready to start writing infrastructure as code directly from your Mac.
