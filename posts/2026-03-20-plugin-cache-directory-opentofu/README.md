# How to Use Plugin Cache Directory in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Performance, Plugin Cache, Provider, Infrastructure as Code, DevOps

Description: A detailed guide to configuring and managing OpenTofu's plugin cache directory, including setup methods, cache key strategies, and troubleshooting stale cache issues.

## Introduction

The plugin cache directory tells OpenTofu where to store downloaded provider binaries. When the same provider version is needed again, OpenTofu symlinks from the cache instead of re-downloading, making `tofu init` nearly instant for cached providers.

## Method 1: Environment Variable

```bash
# Set for the current session
export TF_PLUGIN_CACHE_DIR="/home/ci/.terraform.d/plugin-cache"
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# Verify
tofu init   # Should show "Reusing previously-installed hashicorp/aws" on second run
```

## Method 2: CLI Configuration File

```hcl
# ~/.terraformrc (Linux/macOS) or %APPDATA%/terraform.rc (Windows)
plugin_cache_dir = "/home/ci/.terraform.d/plugin-cache"

# Can also use a relative path to the home directory
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

## Method 3: Per-Project Configuration

For projects that should use a local cache within the repository:

```bash
# Create a project-local cache
mkdir -p .terraform.d/plugin-cache

# Set the cache dir for this project only
export TF_PLUGIN_CACHE_DIR="$(pwd)/.terraform.d/plugin-cache"

# Add to .gitignore
echo ".terraform.d/" >> .gitignore
```

## Cache Invalidation

The cache is keyed on provider source, version, and platform. A new version automatically downloads to a new cache subdirectory:

```bash
# Cache structure — each version is separate
plugin-cache/
└── registry.opentofu.org/hashicorp/aws/
    ├── 5.38.0/linux_amd64/  ← old version
    └── 5.40.0/linux_amd64/  ← new version (downloaded automatically)
```

## Cleaning Up Old Versions

```bash
# Remove all cached versions of the AWS provider older than 5.40.0
CACHE_DIR="$HOME/.terraform.d/plugin-cache/registry.opentofu.org/hashicorp/aws"
ls "$CACHE_DIR"
# 5.38.0  5.39.0  5.40.0

# Remove old versions
rm -rf "$CACHE_DIR/5.38.0" "$CACHE_DIR/5.39.0"

# Or remove all versions and let them re-download fresh
rm -rf "$CACHE_DIR"
```

## Plugin Cache May Dir

The `plugin_cache_may_break_dependency_lock_file` flag relaxes the lock file check when using the cache:

```hcl
# ~/.terraformrc
plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
# Allow cache to be used even if checksums differ from the lock file
# (useful in some corporate mirror setups)
plugin_cache_may_break_dependency_lock_file = true
```

## Troubleshooting Stale Cache

```bash
# If init reports "Failed to retrieve provider ... from cache"
# The cache may have a corrupted binary — delete the specific version

PROVIDER_CACHE="$HOME/.terraform.d/plugin-cache/registry.opentofu.org/hashicorp/aws/5.40.0"
rm -rf "$PROVIDER_CACHE"

# Re-run init to download fresh
tofu init
```

## Verifying Cache is Being Used

```bash
TF_LOG=DEBUG tofu init 2>&1 | grep -i "cache\|reusing"
# Look for: "Reusing previously-installed hashicorp/aws v5.40.0"
# vs: "Installing hashicorp/aws v5.40.0" (downloading fresh)
```

## Conclusion

The plugin cache directory is a simple configuration that pays dividends across every `tofu init` after the first. Configure it via `TF_PLUGIN_CACHE_DIR` in CI/CD, persist it between pipeline runs with a cache action, and periodically prune old versions to keep the cache lean. In air-gapped environments, pre-populate the cache directory and ship it as part of your CI runner image.
