# How to Use the Provider Plugin Cache in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider

Description: Learn how to configure the OpenTofu provider plugin cache to share downloaded providers across multiple configurations and speed up initialization.

## Introduction

By default, OpenTofu downloads provider binaries into each project's `.terraform/providers/` directory. When you work with multiple configurations that use the same providers, this means repeated downloads. The plugin cache stores providers in a shared location so each version is downloaded only once.

## Configuring the Plugin Cache Directory

Set `plugin_cache_dir` in your `.tofurc` file:

```hcl
# ~/.tofurc

plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
```

Or use the environment variable:

```bash
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
```

## Creating the Cache Directory

The directory must exist before OpenTofu can use it:

```bash
mkdir -p ~/.terraform.d/plugin-cache
```

## How the Cache Works

When `tofu init` runs:

1. OpenTofu checks if the required provider version exists in the cache directory
2. If found: creates a symlink from `.terraform/providers/` to the cache entry
3. If not found: downloads the provider, stores it in the cache, then symlinks

```text
~/.terraform.d/plugin-cache/
└── registry.opentofu.org/
    └── hashicorp/
        └── aws/
            └── 5.31.0/
                └── linux_amd64/
                    └── terraform-provider-aws_v5.31.0_x5
```

## Benefits

- Faster `tofu init` for existing provider versions
- Reduced bandwidth consumption
- Shared providers across many workspace directories
- Useful in CI/CD when the cache directory is persisted between runs

## CI/CD Cache Configuration

```yaml
# GitHub Actions with caching
- name: Configure plugin cache
  run: |
    mkdir -p ~/.terraform.d/plugin-cache
    echo 'plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"' > ~/.tofurc

- name: Cache OpenTofu plugins
  uses: actions/cache@v4
  with:
    path: ~/.terraform.d/plugin-cache
    key: tofu-providers-${{ runner.os }}-${{ hashFiles('**/.terraform.lock.hcl') }}
    restore-keys: |
      tofu-providers-${{ runner.os }}-

- name: OpenTofu init
  run: tofu init
```

## Plugin Cache with Multiple Configurations

```bash
# Project A and Project B share the same aws 5.31.0 binary
cd ~/projects/project-a
tofu init  # Downloads aws 5.31.0 to cache

cd ~/projects/project-b
tofu init  # Reuses aws 5.31.0 from cache - instant!
```

## Cache and Lock File Interaction

The cache is separate from the lock file. The lock file records which version to use; the cache stores the binary for that version. Always commit the lock file but never commit the cache directory.

```text
# .gitignore
.terraform/
# Do NOT add ~/.terraform.d/plugin-cache - it lives outside the repo
```

## Limitations

- The cache does not automatically expire old versions; clean up manually if disk space is a concern.
- On some systems, symlinks may not work across different filesystems. Use the same filesystem for the cache and project directories.

## Conclusion

The provider plugin cache eliminates redundant downloads by sharing provider binaries across all your OpenTofu configurations. Configure it once in `.tofurc`, persist the directory between CI runs, and enjoy faster initialization times especially in environments where the same providers are used across many configurations.
