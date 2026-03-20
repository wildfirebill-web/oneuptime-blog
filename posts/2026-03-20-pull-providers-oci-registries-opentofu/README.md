# How to Pull Providers from OCI Registries with OpenTofu

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, OCI Registry, Provider Installation, Containers, Infrastructure

Description: Learn how to configure OpenTofu to pull provider plugins directly from OCI-compatible registries, enabling offline workflows and centralized provider distribution.

## Introduction

OpenTofu 1.8+ can install providers directly from OCI registries using the `oci_mirror` provider installation method. This enables teams to distribute providers through existing container registry infrastructure (ECR, ACR, GCR, GHCR, or any OCI-compliant registry) without running a separate Terraform registry server.

## Configuring OCI Provider Installation

```hcl
# ~/.terraform.rc or set via TF_CLI_CONFIG_FILE

provider_installation {
  # Pull providers from OCI registry
  oci_mirror {
    # OCI registry base URL
    url = "oci://registry.internal.company.com/opentofu-providers"

    # Which providers to pull from OCI (pattern matching)
    include = [
      "registry.opentofu.org/hashicorp/*",
      "registry.opentofu.org/mycompany/*"
    ]
  }

  # Fallback for providers not in OCI registry
  direct {
    exclude = [
      "registry.opentofu.org/hashicorp/*",
      "registry.opentofu.org/mycompany/*"
    ]
  }
}
```

## Authentication Configuration

```hcl
# ~/.terraform.rc - add credentials for the OCI registry

credentials "registry.internal.company.com" {
  # For registries that use token auth
  token = "your-registry-token"
}

# Or use environment variables recognized by Docker/oras:

# DOCKER_USERNAME, DOCKER_PASSWORD
# AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY (for ECR)
```

```bash
# For ECR: configure Docker credential helper
# Install amazon-ecr-credential-helper
cat > ~/.docker/config.json << 'EOF'
{
  "credHelpers": {
    "123456789.dkr.ecr.us-east-1.amazonaws.com": "ecr-login"
  }
}
EOF

# OpenTofu uses the Docker credential store when pulling from OCI
```

## Pulling Providers from GitHub Container Registry

```hcl
# ~/.terraform.rc
credentials "ghcr.io" {
  token = "ghp_yourGitHubPersonalAccessToken"
}

provider_installation {
  oci_mirror {
    url     = "oci://ghcr.io/myorg/opentofu-providers"
    include = ["registry.opentofu.org/mycompany/*"]
  }

  direct {
    exclude = ["registry.opentofu.org/mycompany/*"]
  }
}
```

```hcl
# OpenTofu configuration using OCI-backed provider
terraform {
  required_providers {
    myprovider = {
      source  = "mycompany/myprovider"
      version = "~> 1.0"
    }
  }
}
```

## Pulling Mirrored Public Providers from OCI

```bash
# Script to mirror public providers to OCI registry
#!/bin/bash

set -euo pipefail

REGISTRY="123456789.dkr.ecr.us-east-1.amazonaws.com"
MIRROR_PATH="opentofu-providers"
PROVIDER_MIRROR_DIR="/tmp/provider-mirror"

# Providers to mirror
declare -A PROVIDERS=(
  ["hashicorp/aws"]="5.20.1"
  ["hashicorp/kubernetes"]="2.23.0"
  ["hashicorp/helm"]="2.11.0"
)

# Login to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin "$REGISTRY"

for PROVIDER_PATH in "${!PROVIDERS[@]}"; do
  VERSION="${PROVIDERS[$PROVIDER_PATH]}"
  NAMESPACE=$(echo "$PROVIDER_PATH" | cut -d'/' -f1)
  TYPE=$(echo "$PROVIDER_PATH" | cut -d'/' -f2)

  echo "Mirroring $PROVIDER_PATH@$VERSION..."

  # Download from official registry using tofu providers mirror
  mkdir -p "$PROVIDER_MIRROR_DIR/$PROVIDER_PATH"
  cat > /tmp/mirror-config/main.tf << EOF
terraform {
  required_providers {
    provider = {
      source  = "$PROVIDER_PATH"
      version = "= $VERSION"
    }
  }
}
EOF
  cd /tmp/mirror-config
  tofu init -backend=false
  tofu providers mirror \
    -platform=linux_amd64 \
    -platform=linux_arm64 \
    "$PROVIDER_MIRROR_DIR/"

  # Push to OCI using oras
  cd "$PROVIDER_MIRROR_DIR/registry.opentofu.org/$NAMESPACE/$TYPE/"

  oras push "${REGISTRY}/${MIRROR_PATH}/${NAMESPACE}-${TYPE}:${VERSION}" \
    terraform-provider-${TYPE}_${VERSION}_linux_amd64.zip:application/vnd.opentofu.provider.v1.linux.amd64 \
    terraform-provider-${TYPE}_${VERSION}_linux_arm64.zip:application/vnd.opentofu.provider.v1.linux.arm64 \
    terraform-provider-${TYPE}_${VERSION}_SHA256SUMS:application/vnd.opentofu.provider.v1.shasums

  echo "  Pushed: ${REGISTRY}/${MIRROR_PATH}/${NAMESPACE}-${TYPE}:${VERSION}"
done

echo "All providers mirrored to OCI registry"
```

## Verifying OCI Provider Pull

```bash
# Check that OpenTofu is pulling from OCI
TF_LOG=DEBUG tofu init 2>&1 | grep -i "oci\|mirror\|pull"

# Expected output shows OCI pull requests
# [DEBUG] provider install: oci://registry.../hashicorp-aws:5.20.1

# Verify locally with oras
oras manifest fetch 123456789.dkr.ecr.us-east-1.amazonaws.com/opentofu-providers/hashicorp-aws:5.20.1
```

## Fully Offline OCI Configuration

```hcl
# ~/.terraform.rc - Completely offline, all providers from OCI

provider_installation {
  oci_mirror {
    url     = "oci://registry.internal.company.com/opentofu-providers"
    include = ["registry.opentofu.org/*/*"]
  }

  # No direct fallback - fully offline
}
```

```bash
# Test offline init
export TF_CLI_CONFIG_FILE=/etc/opentofu/terraform.rc
# Disconnect from internet or block registry.opentofu.org in /etc/hosts

tofu init  # Should succeed using OCI registry only
```

## CI/CD Integration

```yaml
# .github/workflows/deploy.yml
- name: Configure OpenTofu OCI mirror
  run: |
    mkdir -p ~/.terraform.d
    cat > ~/.terraform.rc << 'EOF'
    credentials "ghcr.io" {
      token = "${{ secrets.GITHUB_TOKEN }}"
    }
    provider_installation {
      oci_mirror {
        url     = "oci://ghcr.io/${{ github.repository_owner }}/opentofu-providers"
        include = ["registry.opentofu.org/hashicorp/*"]
      }
      direct {
        exclude = ["registry.opentofu.org/hashicorp/*"]
      }
    }
    EOF

- name: OpenTofu Init
  run: tofu init
  env:
    TF_CLI_CONFIG_FILE: ~/.terraform.rc
```

## Conclusion

Pulling providers from OCI registries requires two things: the OCI artifact must be pushed with the correct content-type annotations (using `oras`), and the `oci_mirror` block in `terraform.rc` must point to the registry URL. OpenTofu uses Docker credential helpers for OCI authentication, so any registry that Docker can authenticate with works. This approach is ideal for air-gapped environments already running container registry infrastructure, eliminating the need for a separate provider distribution server.
