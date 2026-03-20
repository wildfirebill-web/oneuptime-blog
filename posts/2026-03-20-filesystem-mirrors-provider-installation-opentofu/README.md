# How to Use Filesystem Mirrors for Provider Installation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Terraform, Provider Mirror, Filesystem Mirror, Air-Gapped, Infrastructure as Code

Description: Learn how to configure OpenTofu filesystem mirrors to install providers from a local directory in air-gapped environments or to speed up provider installation in CI/CD pipelines.

---

OpenTofu normally downloads providers from the Terraform Registry or OpenTofu Registry over the internet. In air-gapped environments, restricted networks, or for CI/CD performance, you can configure a filesystem mirror to serve providers from a local directory.

---

## What is a Filesystem Mirror?

A filesystem mirror is a local directory that contains pre-downloaded provider packages in the format OpenTofu expects. When configured, OpenTofu checks the mirror first before hitting the internet.

---

## Mirror Directory Structure

OpenTofu expects a specific directory layout:

```text
mirror/
└── registry.opentofu.org/
    └── hashicorp/
        └── aws/
            ├── 5.31.0/
            │   ├── terraform-provider-aws_5.31.0_linux_amd64.zip
            │   └── terraform-provider-aws_5.31.0_darwin_arm64.zip
            └── 5.32.0/
                └── terraform-provider-aws_5.32.0_linux_amd64.zip
```

---

## Creating a Filesystem Mirror

### Method 1: Using tofu providers mirror

```bash
# Create mirror directory

mkdir -p /opt/opentofu/providers

# Download providers into the mirror
cd /path/to/your/project
tofu providers mirror /opt/opentofu/providers

# For specific platforms only
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  /opt/opentofu/providers
```

### Method 2: Manual Download

```bash
# Download AWS provider for Linux AMD64
PROVIDER_VERSION="5.31.0"
MIRROR_DIR="/opt/opentofu/providers"

mkdir -p "$MIRROR_DIR/registry.opentofu.org/hashicorp/aws/$PROVIDER_VERSION"

curl -L "https://releases.hashicorp.com/terraform-provider-aws/${PROVIDER_VERSION}/terraform-provider-aws_${PROVIDER_VERSION}_linux_amd64.zip" \
  -o "$MIRROR_DIR/registry.opentofu.org/hashicorp/aws/$PROVIDER_VERSION/terraform-provider-aws_${PROVIDER_VERSION}_linux_amd64.zip"
```

---

## Configuring OpenTofu to Use the Mirror

### User-Level Configuration (~/.terraformrc or ~/.tofurc)

```hcl
# ~/.tofurc
provider_installation {
  filesystem_mirror {
    path    = "/opt/opentofu/providers"
    include = ["registry.opentofu.org/*/*"]
  }

  # Fall back to direct registry for anything not in mirror
  direct {
    exclude = ["registry.opentofu.org/hashicorp/*"]
  }
}
```

### Project-Level Configuration

```hcl
# .terraform.lock.hcl is project-specific
# Use TF_CLI_CONFIG_FILE for project-level override:
export TF_CLI_CONFIG_FILE="$PWD/.tofurc"
```

### Mirror Only (No Internet Fallback)

```hcl
# ~/.tofurc - strict air-gapped mode
provider_installation {
  filesystem_mirror {
    path    = "/opt/opentofu/providers"
    include = ["registry.opentofu.org/*/*"]
  }
  # No direct {} block = no internet fallback
}
```

---

## CI/CD Pipeline Integration

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

env:
  TF_CLI_CONFIG_FILE: /opt/opentofu/.tofurc

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v4

      - name: Sync provider mirror (daily via cron)
        run: |
          tofu providers mirror /opt/opentofu/providers

      - name: Setup OpenTofu
        uses: opentofu/setup-opentofu@v1

      - name: Initialize (uses mirror)
        run: tofu init

      - name: Apply
        run: tofu apply -auto-approve
```

---

## Network Mirror Alternative

For organizations with a central repository server (Artifactory, Nexus):

```hcl
# ~/.tofurc
provider_installation {
  network_mirror {
    url     = "https://artifactory.corp.example.com/opentofu-providers/"
    include = ["registry.opentofu.org/hashicorp/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/hashicorp/*"]
  }
}
```

---

## Verify Mirror is Being Used

```bash
# Initialize with extra verbosity
TF_LOG=DEBUG tofu init 2>&1 | grep -i "mirror\|installing"

# Expected output:
# - Installing hashicorp/aws v5.31.0 from filesystem mirror at /opt/opentofu/providers
```

---

## Lock File and Mirror

After initializing with a mirror, the `.terraform.lock.hcl` file records provider hashes:

```bash
# Initialize and create lock file
tofu init

# Lock file records:
# - exact version
# - hash of provider binary
# Subsequent inits validate against this lock file
```

---

## Updating the Mirror

```bash
# Add new provider versions to the mirror
tofu providers mirror /opt/opentofu/providers

# Or update for a specific provider
cd /path/to/project-needing-newer-version
tofu providers mirror /opt/opentofu/providers
```

---

## Best Practices

1. **Build mirrors in CI/CD** periodically - providers are versioned and don't change
2. **Use include/exclude** to mix mirror and direct for specific providers
3. **Commit the mirror to a shared artifact store** for team and CI sharing
4. **Keep the mirror read-only** during deploys to prevent accidental modification
5. **Version your mirror** to align with `.terraform.lock.hcl` pinned versions

---

## Conclusion

OpenTofu filesystem mirrors enable provider installation in air-gapped environments and significantly speed up CI/CD pipelines by eliminating registry downloads on every run. Configure the mirror once, populate it with `tofu providers mirror`, and reference it in your `.tofurc` configuration.

---

*Manage your infrastructure securely with [OneUptime](https://oneuptime.com) - monitoring and observability for your deployments.*
