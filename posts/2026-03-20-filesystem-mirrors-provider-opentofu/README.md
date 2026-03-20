# How to Use Filesystem Mirrors for Provider Installation in OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Filesystem Mirror, Provider Installation, Offline, Configuration

Description: Learn how to configure filesystem mirrors in OpenTofu to install providers from a local directory instead of the public registry, enabling faster installs and offline workflows.

## Introduction

OpenTofu's `filesystem_mirror` configuration in the CLI configuration file redirects provider downloads from the public registry to a local directory. This is the most direct way to achieve offline provider installation — no server required, just a directory with the right structure.

## CLI Configuration File Location

```bash
# Default locations for the CLI configuration file:
# Unix/Linux/macOS: ~/.terraform.rc or ~/.terraformrc
# Windows: %APPDATA%/terraform.rc

# Override with environment variable
export TF_CLI_CONFIG_FILE=/etc/opentofu/terraform.rc

# Verify the file is recognized
tofu version  # Shows the config file path if set
```

## Basic Filesystem Mirror Configuration

```hcl
# ~/.terraform.rc

provider_installation {
  filesystem_mirror {
    path = "/opt/opentofu/provider-mirror"
  }
}
```

```bash
# The path must contain providers in the expected structure:
# <mirror-path>/<hostname>/<namespace>/<type>/<version>/<os>_<arch>/
# Example:
/opt/opentofu/provider-mirror/
└── registry.opentofu.org/
    └── hashicorp/
        └── aws/
            └── 5.20.1/
                ├── terraform-provider-aws_5.20.1_linux_amd64.zip
                └── terraform-provider-aws_5.20.1_darwin_arm64.zip
```

## Populating the Mirror

```bash
# Use tofu providers mirror to populate the directory

# Create a configuration with all needed providers
cat > /tmp/provider-config/versions.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.20.1"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "= 2.23.0"
    }
    vault = {
      source  = "hashicorp/vault"
      version = "= 3.20.0"
    }
  }
}
EOF

cd /tmp/provider-config
tofu init

# Mirror to local directory (downloads current platform)
tofu providers mirror /opt/opentofu/provider-mirror/

# Mirror for specific platforms
tofu providers mirror \
  -platform=linux_amd64 \
  -platform=darwin_arm64 \
  /opt/opentofu/provider-mirror/
```

## Include and Exclude Patterns

```hcl
# ~/.terraform.rc

provider_installation {
  # Mirror for specific providers
  filesystem_mirror {
    path    = "/opt/opentofu/provider-mirror"
    include = [
      "registry.opentofu.org/hashicorp/*",
      "registry.opentofu.org/datadog/datadog"
    ]
  }

  # Fall back to direct download for other providers
  direct {
    exclude = [
      "registry.opentofu.org/hashicorp/*",
      "registry.opentofu.org/datadog/datadog"
    ]
  }
}
```

## Multiple Mirror Directories

```hcl
# ~/.terraform.rc

provider_installation {
  # Company-approved providers (highest priority)
  filesystem_mirror {
    path    = "/opt/opentofu/approved-providers"
    include = ["registry.opentofu.org/hashicorp/*"]
  }

  # Community providers
  filesystem_mirror {
    path    = "/opt/opentofu/community-providers"
    include = ["registry.opentofu.org/*/*"]
  }

  # No direct downloads allowed
  direct {
    exclude = ["registry.opentofu.org/*/*"]
  }
}
```

## Verifying Mirror Contents

```bash
# Check available providers and versions in mirror
find /opt/opentofu/provider-mirror -name "*.zip" | sort

# Check the JSON index files
find /opt/opentofu/provider-mirror -name "*.json" | sort

# Expected output for a correctly populated mirror:
# /opt/opentofu/provider-mirror/registry.opentofu.org/hashicorp/aws/5.20.1.json
# /opt/opentofu/provider-mirror/registry.opentofu.org/hashicorp/aws/terraform-provider-aws_5.20.1_linux_amd64.zip

# Validate a specific provider zip
unzip -t /opt/opentofu/provider-mirror/registry.opentofu.org/hashicorp/aws/terraform-provider-aws_5.20.1_linux_amd64.zip
```

## Checking Provider Signatures

```bash
# Providers in the mirror include signature verification
# Check the SHA256SUMS file
cat /opt/opentofu/provider-mirror/registry.opentofu.org/hashicorp/aws/5.20.1.json

# Expected JSON structure:
# {
#   "versions": {
#     "5.20.1": {
#       "protocols": ["5.0"],
#       "platforms": [
#         {
#           "os": "linux",
#           "arch": "amd64",
#           "filename": "terraform-provider-aws_5.20.1_linux_amd64.zip",
#           "shasum": "abc123..."
#         }
#       ]
#     }
#   }
# }
```

## CI/CD Pipeline Configuration

```yaml
# .github/workflows/opentofu.yml
- name: Configure OpenTofu mirror
  run: |
    mkdir -p ~/.terraform.d
    cat > ~/.terraform.rc << 'EOF'
    provider_installation {
      filesystem_mirror {
        path    = "/opt/opentofu/provider-mirror"
        include = ["registry.opentofu.org/*/*"]
      }
      direct {
        exclude = ["registry.opentofu.org/*/*"]
      }
    }
    EOF

- name: OpenTofu Init
  run: tofu init
  env:
    TF_CLI_CONFIG_FILE: ~/.terraform.rc
```

## Sharing Mirror via NFS

```bash
# Mount provider mirror from NFS in enterprise environments

# /etc/fstab entry
nas.company.com:/exports/opentofu-providers /opt/opentofu/provider-mirror nfs ro,noatime 0 0

# All machines mount the same read-only mirror
# Configure ~/.terraform.rc to use /opt/opentofu/provider-mirror
```

## Conclusion

Filesystem mirrors in OpenTofu work by matching the provider's registry hostname and path in the mirror directory structure. The `tofu providers mirror` command handles the directory structure creation and downloads providers in the correct format including JSON metadata and SHA256 signatures. Use `include` patterns to send only specific providers to the mirror, with a `direct` fallback for providers not in the mirror. For fully offline environments, exclude all providers from `direct` to ensure nothing tries to reach the internet.
