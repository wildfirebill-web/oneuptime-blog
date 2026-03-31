# Using tofu providers lock in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, IaC, Provider, Lock

Description: Learn how to use the tofu providers lock command to manage provider checksums for multiple platforms in your lock file.

The `tofu providers lock` command updates the `.terraform.lock.hcl` file with checksums for the current provider selections. It's particularly useful when you need to add checksums for platforms you're not currently developing on - like adding Linux checksums from a macOS development machine.

## Basic Usage

```bash
# Update lock file with checksums for current platform

tofu providers lock

# This reads your required_providers and updates .terraform.lock.hcl
# with the current provider versions and their checksums
```

## Adding Checksums for Multiple Platforms

```bash
# Add checksums for all platforms your team uses
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_amd64 \
  -platform=darwin_arm64 \
  -platform=windows_amd64
```

This is essential when:
- Your team uses a mix of macOS and Linux
- CI/CD runs on Linux but developers use macOS
- You want to support multiple architectures

## Platform Identifiers

```bash
# Common platform identifiers
linux_amd64     # Most CI/CD systems, Intel/AMD Linux
linux_arm64     # ARM Linux (AWS Graviton, Raspberry Pi)
darwin_amd64    # Intel Macs
darwin_arm64    # Apple Silicon Macs (M1, M2, M3)
windows_amd64   # Windows x64
freebsd_amd64   # FreeBSD
```

## Lock File Result

After running `tofu providers lock -platform=linux_amd64 -platform=darwin_arm64`:

```hcl
# .terraform.lock.hcl
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.38.0"
  constraints = "~> 5.0"
  hashes = [
    # Hashes for linux_amd64
    "h1:abc123def456...",
    "zh:789ghi012jkl...",
    
    # Hashes for darwin_arm64
    "h1:mno345pqr678...",
    "zh:stu901vwx234...",
  ]
}
```

## Using providers lock with a Mirror

```bash
# Lock providers from a filesystem mirror
tofu providers lock \
  -fs-mirror=/opt/terraform-mirror \
  -platform=linux_amd64 \
  -platform=darwin_arm64

# Lock from a network mirror
tofu providers lock \
  -net-mirror=https://mirror.example.com/ \
  -platform=linux_amd64
```

## Locking Specific Providers

```bash
# Lock only specific providers (not all in configuration)
tofu providers lock \
  -platform=linux_amd64 \
  registry.opentofu.org/hashicorp/aws \
  registry.opentofu.org/hashicorp/google

# Other providers in the configuration are not touched
```

## Common Workflow: Cross-Platform Team

```bash
# Developer on macOS (M1) creates initial lock file
tofu init    # Generates lock file for darwin_arm64

# Add checksums for Linux CI/CD
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64

# Commit the updated lock file
git add .terraform.lock.hcl
git commit -m "Add Linux platform checksums to provider lock file"
```

## Verifying the Lock File

```bash
# Check lock file contents
cat .terraform.lock.hcl

# Verify tofu accepts the lock file
tofu init

# If you see this, the lock file is valid:
# - Installed hashicorp/aws v5.38.0 (unauthenticated)
# or
# - hashicorp/aws: using previously-installed v5.38.0
```

## When to Run providers lock

```bash
# Run providers lock when:
# 1. Setting up a new project
tofu init
tofu providers lock -platform=linux_amd64 -platform=darwin_arm64

# 2. Upgrading providers
tofu init -upgrade
tofu providers lock -platform=linux_amd64 -platform=darwin_arm64

# 3. Adding a new provider
# (after updating required_providers)
tofu init
tofu providers lock -platform=linux_amd64 -platform=darwin_arm64

# 4. A new platform needs to be supported
tofu providers lock -platform=linux_arm64
```

## Difference Between init and providers lock

```bash
# tofu init:
# - Downloads providers if not in cache
# - Updates lock file for CURRENT platform only
# - Runs full initialization (modules, backend, etc.)

# tofu providers lock:
# - Only updates lock file checksums
# - Can add checksums for platforms you don't have locally
# - Does NOT download provider binaries
# - Does NOT initialize backends or modules
```

## Conclusion

`tofu providers lock` is essential for teams working across different operating systems and architectures. Run it after any provider version change and add checksums for all platforms your team uses. Committing the resulting lock file ensures every developer, CI/CD pipeline, and production deployment uses identical provider versions with verified checksums.
