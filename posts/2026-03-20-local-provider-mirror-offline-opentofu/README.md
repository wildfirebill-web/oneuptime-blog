# How to Create a Local Provider Mirror for Offline OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Provider Mirror, Offline, Air-Gapped, Infrastructure

Description: Learn how to create and maintain a local provider mirror for OpenTofu to enable offline provider installation and consistent provider versions across your team.

## Introduction

A local provider mirror is a directory (or web server) that contains provider plugin archives in the format OpenTofu expects. It lets teams install providers without internet access, pin exact provider versions, and reduce download times in CI/CD pipelines.

## Creating a Mirror with tofu providers mirror

```bash
# Create a configuration that lists all needed providers
cat > /tmp/mirror-setup/main.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.0"
    }
  }
}
EOF

cd /tmp/mirror-setup
tofu init

# Mirror to local directory
tofu providers mirror /opt/provider-mirror/

# Output shows what was downloaded:
# - registry.opentofu.org/hashicorp/aws 5.20.1
# - registry.opentofu.org/hashicorp/kubernetes 2.23.0
# ...
```

## Mirror Directory Structure

```
/opt/provider-mirror/
└── registry.opentofu.org/
    └── hashicorp/
        ├── aws/
        │   ├── 5.20.1.json          # Version metadata
        │   └── terraform-provider-aws_5.20.1_linux_amd64.zip
        ├── kubernetes/
        │   ├── 2.23.0.json
        │   └── terraform-provider-kubernetes_2.23.0_linux_amd64.zip
        └── random/
            ├── 3.5.1.json
            └── terraform-provider-random_3.5.1_linux_amd64.zip
```

```bash
# Check what's in the mirror
find /opt/provider-mirror -name "*.json" | head -20
find /opt/provider-mirror -name "*.zip" | wc -l
```

## Configuring OpenTofu to Use the Mirror

```hcl
# ~/.terraform.rc (or set TF_CLI_CONFIG_FILE to this file's path)

provider_installation {
  filesystem_mirror {
    path    = "/opt/provider-mirror"
    include = ["registry.opentofu.org/*/*"]
  }

  # Fallback to direct for providers not in mirror
  # Remove this block for fully offline environments
  direct {
    exclude = [
      "registry.opentofu.org/hashicorp/aws",
      "registry.opentofu.org/hashicorp/kubernetes"
    ]
  }
}
```

```bash
# Test the mirror configuration
export TF_CLI_CONFIG_FILE=/etc/opentofu/terraform.rc
tofu init  # Should use mirror, no internet access needed
```

## Multiple Platform Support

```bash
# Mirror providers for multiple operating systems
# Useful when your team uses both macOS and Linux

# Linux amd64 (CI/CD servers)
GOOS=linux GOARCH=amd64 tofu providers mirror /opt/provider-mirror/

# macOS arm64 (Apple Silicon developers)
GOOS=darwin GOARCH=arm64 tofu providers mirror /opt/provider-mirror/

# Windows amd64
GOOS=windows GOARCH=amd64 tofu providers mirror /opt/provider-mirror/

# All platforms in one directory is fine - OpenTofu selects the right zip
```

## Maintaining the Mirror

```bash
#!/bin/bash
# update-mirror.sh - Script to update the provider mirror

set -euo pipefail

MIRROR_DIR="/opt/provider-mirror"
PROVIDERS_CONFIG="/opt/mirror-config/providers.tf"

echo "Updating provider mirror at $MIRROR_DIR"

# Initialize to resolve latest compatible versions
cd "$(dirname "$PROVIDERS_CONFIG")"
tofu init -upgrade

# Update the mirror with new versions
tofu providers mirror "$MIRROR_DIR"

echo "Mirror update complete"
ls -la "$MIRROR_DIR/registry.opentofu.org/hashicorp/"
```

```bash
# Run update on a schedule (cron)
0 2 * * 1 /opt/scripts/update-mirror.sh >> /var/log/mirror-update.log 2>&1
```

## Sharing the Mirror via HTTP

```nginx
# /etc/nginx/sites-enabled/provider-mirror
server {
    listen 443 ssl http2;
    server_name provider-mirror.internal.company.com;

    ssl_certificate     /etc/ssl/certs/mirror.crt;
    ssl_certificate_key /etc/ssl/private/mirror.key;

    root /opt/provider-mirror;

    # Required: serve JSON metadata with correct content type
    location ~* \.json$ {
        add_header Content-Type "application/json";
        add_header Cache-Control "max-age=3600";
    }

    # Provider zip files
    location ~* \.zip$ {
        add_header Content-Type "application/zip";
    }

    location / {
        try_files $uri $uri/ =404;
    }

    # Access log for auditing
    access_log /var/log/nginx/provider-mirror.log;
}
```

```hcl
# Use network mirror in terraform.rc
provider_installation {
  network_mirror {
    url     = "https://provider-mirror.internal.company.com/"
    include = ["registry.opentofu.org/*/*"]
  }
}
```

## Version Locking in the Mirror

```bash
# Lock to exact versions (don't use ~> constraints for mirrors)
cat > /tmp/mirror-setup/versions.tf << 'EOF'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "= 5.20.1"  # Exact version for reproducible mirrors
    }
  }
}
EOF
```

## Mirror for Third-Party Providers

```bash
# Third-party providers follow the same pattern
# Example: DataDog provider

cat > /tmp/mirror-setup/main.tf << 'EOF'
terraform {
  required_providers {
    datadog = {
      source  = "DataDog/datadog"
      version = "= 3.30.0"
    }
  }
}
EOF

tofu init
tofu providers mirror /opt/provider-mirror/
# Creates: /opt/provider-mirror/registry.opentofu.org/datadog/datadog/3.30.0.json
```

## Conclusion

The `tofu providers mirror` command downloads providers into the exact directory structure OpenTofu expects. The `filesystem_mirror` configuration in `.terraform.rc` redirects all provider downloads to that local directory. For teams, host the mirror directory via nginx to give everyone access without copying files manually. Update the mirror periodically by re-running `tofu providers mirror` against a configuration file that lists all needed provider versions.
