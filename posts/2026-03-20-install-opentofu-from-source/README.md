# How to Install OpenTofu from Source

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Source Build, Go, Installation, Infrastructure as Code, DevOps

Description: A comprehensive guide to building and installing OpenTofu from its source code on Linux and macOS.

## Introduction

Building OpenTofu from source gives you the ability to run the latest development builds, contribute to the project, or customize the binary for your specific needs. This guide covers building OpenTofu from source on Linux and macOS.

## Prerequisites

- Go 1.21 or later
- Git
- Make
- 4 GB RAM (for compilation)
- Linux or macOS

## Step 1: Install Go

```bash
# Download Go (replace with latest version)
GO_VERSION="1.22.0"
curl -LO "https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz"

# Install Go
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf "go${GO_VERSION}.linux-amd64.tar.gz"

# Add Go to PATH
export PATH=$PATH:/usr/local/go/bin
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verify Go installation
go version
```

## Step 2: Clone the OpenTofu Repository

```bash
# Create a workspace directory
mkdir -p ~/go/src/github.com/opentofu
cd ~/go/src/github.com/opentofu

# Clone the OpenTofu repository
git clone https://github.com/opentofu/opentofu.git
cd opentofu
```

## Step 3: Checkout a Specific Version

```bash
# List available tags
git tag -l | sort -V | tail -20

# Checkout a specific release (recommended)
git checkout v1.9.0

# Or stay on main for the latest development build
git checkout main
```

## Step 4: Build OpenTofu

```bash
# Navigate to the repository root
cd ~/go/src/github.com/opentofu/opentofu

# Build using Go directly
go build -o tofu ./cmd/tofu

# Or use Make if available
make build

# The binary will be at ./tofu
```

## Step 5: Install the Binary

```bash
# Move the binary to a directory in PATH
sudo mv tofu /usr/local/bin/
sudo chmod +x /usr/local/bin/tofu

# Verify the installation
tofu version
```

## Building for Different Platforms (Cross-Compilation)

```bash
# Build for Linux AMD64
GOOS=linux GOARCH=amd64 go build -o tofu-linux-amd64 ./cmd/tofu

# Build for macOS ARM64 (Apple Silicon)
GOOS=darwin GOARCH=arm64 go build -o tofu-darwin-arm64 ./cmd/tofu

# Build for Windows AMD64
GOOS=windows GOARCH=amd64 go build -o tofu-windows-amd64.exe ./cmd/tofu
```

## Running Tests

```bash
# Run unit tests
go test ./...

# Run specific package tests
go test ./internal/command/...

# Run with verbose output
go test -v ./internal/configs/...
```

## Building with Version Information

```bash
# Build with version metadata embedded
BUILD_VERSION=$(git describe --tags --always)
BUILD_DATE=$(date -u +%Y-%m-%dT%H:%M:%SZ)
BUILD_COMMIT=$(git rev-parse --short HEAD)

go build \
  -ldflags "-X github.com/opentofu/opentofu/version.dev=no -X github.com/opentofu/opentofu/version.Version=${BUILD_VERSION}" \
  -o tofu \
  ./cmd/tofu

# Check version info
./tofu version
```

## Verifying the Build

```hcl
# test.tf
terraform {
  required_version = ">= 1.6"
}

output "source_build" {
  value = "OpenTofu built from source is working!"
}
```

```bash
tofu init
tofu apply -auto-approve
```

## Keeping Your Build Up to Date

```bash
# Pull latest changes
cd ~/go/src/github.com/opentofu/opentofu
git fetch --tags
git pull origin main

# Rebuild
go build -o tofu ./cmd/tofu
sudo mv tofu /usr/local/bin/
tofu version
```

## Conclusion

Building OpenTofu from source provides maximum control and flexibility. Whether you need the latest features before a release, want to contribute to the project, or need to customize the build for your environment, the source build process is straightforward for anyone familiar with Go. The official GitHub repository provides all the tools needed to build a production-ready binary.
