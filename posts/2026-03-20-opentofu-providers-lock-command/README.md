# How to Pre-Populate Provider Lock File Checksums with tofu providers lock

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Infrastructure as Code, Provider

Description: Learn how to use the tofu providers lock command to add checksums for multiple platforms to your lock file without running tofu init on each platform.

## Introduction

The `tofu providers lock` command updates the `.terraform.lock.hcl` file with checksums for specified platforms. This is essential when your team develops on macOS but deploys from Linux CI/CD - the lock file needs checksums for both platforms, and `tofu init` only adds checksums for the current platform.

## Basic Usage

```bash
# Add checksums for the current platform

tofu providers lock

# Add checksums for specific platforms
tofu providers lock \
  -platform=linux_amd64 \
  -platform=linux_arm64 \
  -platform=darwin_arm64 \
  -platform=darwin_amd64 \
  -platform=windows_amd64
```

## Why This is Needed

When a developer runs `tofu init` on macOS:

```hcl
# .terraform.lock.hcl - only darwin checksums initially
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:mac-checksum...",
    "zh:darwin_arm64-checksum...",
  ]
}
```

When CI (Linux) runs `tofu init` and there are no Linux checksums, it adds them. This causes an unexpected lock file change in CI. Running `tofu providers lock -platform=linux_amd64` upfront prevents this.

## Adding Multiple Platform Checksums

```bash
# Run from your project directory after initial tofu init
tofu providers lock \
  -platform=linux_amd64 \
  -platform=darwin_arm64

# Commit the updated lock file
git add .terraform.lock.hcl
git commit -m "Add Linux checksums to lock file"
```

## Using a Network Mirror

```bash
# Add checksums sourced from a mirror
tofu providers lock \
  -net-mirror=https://mirror.acme-corp.com/providers/ \
  -platform=linux_amd64
```

## Using a Filesystem Mirror

```bash
# Add checksums from a local mirror
tofu providers lock \
  -fs-mirror=/opt/terraform-providers/ \
  -platform=linux_amd64
```

## CI/CD Integration

Add a lock file verification step to catch unapproved changes:

```yaml
# .github/workflows/validate.yml
- name: Check lock file is up to date
  run: |
    tofu providers lock \
      -platform=linux_amd64 \
      -platform=darwin_arm64

    if ! git diff --exit-code .terraform.lock.hcl; then
      echo "Lock file is out of date. Run 'tofu providers lock' and commit the changes."
      exit 1
    fi
```

## Lock File After providers lock

After running `tofu providers lock -platform=linux_amd64 -platform=darwin_arm64`:

```hcl
provider "registry.opentofu.org/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:...",
    "zh:linux_amd64_checksum...",
    "zh:darwin_arm64_checksum...",
  ]
}
```

Both platform checksums are now present, ensuring any platform can initialize without modifying the lock file.

## Conclusion

`tofu providers lock` is the tool for keeping lock file checksums complete across all deployment platforms. Run it after adding providers or changing versions, specifying all platforms your team and CI/CD pipelines use. This prevents unexpected lock file mutations in CI and ensures the lock file is a complete, verified record of provider checksums.
