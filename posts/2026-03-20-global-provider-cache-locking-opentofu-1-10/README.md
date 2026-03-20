# How to Use Global Provider Cache Locking Introduced in OpenTofu 1.10

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Cache, OpenTofu 1.10, Performance, Infrastructure as Code

Description: Learn how to use global provider cache locking introduced in OpenTofu 1.10 to safely share a provider cache across concurrent OpenTofu invocations.

## Introduction

OpenTofu 1.10 introduced file-based locking for the global provider cache directory (`TF_PLUGIN_CACHE_DIR`). Previously, running multiple `tofu init` operations simultaneously with a shared cache could cause corruption. With cache locking, concurrent initializations safely share a single cache directory without conflicts.

## Setting Up a Global Provider Cache

Configure the shared provider cache directory.

```bash
# Set the global cache directory

export TF_PLUGIN_CACHE_DIR="$HOME/.tofu/plugin-cache"

# Create the directory if it doesn't exist
mkdir -p "$TF_PLUGIN_CACHE_DIR"

# Add to your shell profile for persistence
echo 'export TF_PLUGIN_CACHE_DIR="$HOME/.tofu/plugin-cache"' >> ~/.zshrc
```

## Configuring via .tofurc

Set the cache directory in the OpenTofu CLI configuration file.

```hcl
# ~/.tofurc
plugin_cache_dir = "/home/ci/.tofu/plugin-cache"
```

## Why Cache Locking Matters in CI/CD

Without locking, parallel jobs downloading the same provider could corrupt the cache. With 1.10, concurrent jobs serialize access safely.

```yaml
# .github/workflows/tofu-parallel.yml
jobs:
  plan-prod:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup OpenTofu with shared cache
        run: |
          mkdir -p ~/.tofu/plugin-cache
          echo 'plugin_cache_dir = "/root/.tofu/plugin-cache"' > ~/.tofurc
      - run: tofu init   # safely concurrent with plan-staging

  plan-staging:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Setup OpenTofu with shared cache
        run: |
          mkdir -p ~/.tofu/plugin-cache
          echo 'plugin_cache_dir = "/root/.tofu/plugin-cache"' > ~/.tofurc
      - run: tofu init   # safely concurrent with plan-prod
```

## Caching Providers in Docker Builds

Use a mounted cache volume in Docker-based CI.

```dockerfile
# Dockerfile for CI runner
FROM alpine:3.19

RUN apk add --no-cache curl unzip

# Install OpenTofu
RUN curl -Lo /tmp/tofu.zip \
  "https://github.com/opentofu/opentofu/releases/download/v1.10.0/tofu_1.10.0_linux_amd64.zip" && \
  unzip /tmp/tofu.zip -d /usr/local/bin/ && \
  rm /tmp/tofu.zip

# Create cache directory
RUN mkdir -p /cache/tofu/plugins
ENV TF_PLUGIN_CACHE_DIR=/cache/tofu/plugins

WORKDIR /workspace
```

```bash
# Run with a persistent cache volume
docker run -v tofu-plugin-cache:/cache/tofu/plugins \
  my-ci-runner:latest tofu init
```

## Monitoring Cache Usage

Check what providers are cached and their sizes.

```bash
# View cached providers
ls -lh "$TF_PLUGIN_CACHE_DIR"

# Check total cache size
du -sh "$TF_PLUGIN_CACHE_DIR"

# Find the largest cached providers
du -sh "$TF_PLUGIN_CACHE_DIR"/*/* | sort -hr | head -20

# Example output:
# 147M  /home/user/.tofu/plugin-cache/registry.opentofu.org/hashicorp/aws/5.50.0/linux_amd64
# 52M   /home/user/.tofu/plugin-cache/registry.opentofu.org/hashicorp/azurerm/3.100.0/linux_amd64
# 23M   /home/user/.tofu/plugin-cache/registry.opentofu.org/hashicorp/google/5.25.0/linux_amd64
```

## Combining Cache with Network Mirror

For air-gapped environments, combine the cache with a network mirror.

```hcl
# ~/.tofurc
plugin_cache_dir = "/opt/tofu/plugin-cache"

provider_installation {
  network_mirror {
    url = "https://nexus.internal.example.com/repository/tofu-providers/"
  }
  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
```

## Summary

Global provider cache locking in OpenTofu 1.10 makes shared provider caches safe for concurrent use. Configure `TF_PLUGIN_CACHE_DIR` or `plugin_cache_dir` in `.tofurc` to share downloads across projects and CI jobs. This reduces network traffic, speeds up `tofu init`, and is especially valuable in CI/CD environments where many parallel jobs initialize the same providers.
