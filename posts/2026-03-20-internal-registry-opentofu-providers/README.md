# How to Set Up an Internal Registry for OpenTofu Providers

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Internal Registry, Provider Distribution, Enterprise, Self-Hosted

Description: Learn how to set up an internal provider registry for OpenTofu that serves custom or mirrored providers to your organization using the Terraform Registry Protocol.

## Introduction

An internal OpenTofu provider registry serves two purposes: distributing internally-developed providers and mirroring public providers for air-gapped environments. The Terraform Registry Protocol (which OpenTofu uses) defines the HTTP API that the registry must implement. This guide covers implementing a compliant registry and hosting custom providers.

## Registry Protocol Overview

```
# Service discovery
GET /.well-known/terraform.json

# List provider versions
GET /v1/providers/<namespace>/<type>/versions

# Download metadata for a specific version/platform
GET /v1/providers/<namespace>/<type>/<version>/download/<os>/<arch>
```

## Implementing a Minimal Registry with nginx + Static Files

```bash
# Create registry structure
mkdir -p /var/www/registry/v1/providers/mycompany/

# Create the service discovery file
cat > /var/www/registry/.well-known/terraform.json << 'EOF'
{
  "providers.v1": "/v1/providers/"
}
EOF
```

```bash
# Create provider version index
mkdir -p /var/www/registry/v1/providers/mycompany/myapp

cat > /var/www/registry/v1/providers/mycompany/myapp/versions << 'EOF'
{
  "id": "mycompany/myapp",
  "versions": [
    {
      "version": "1.0.0",
      "protocols": ["5.0"],
      "platforms": [
        {"os": "linux", "arch": "amd64"},
        {"os": "linux", "arch": "arm64"},
        {"os": "darwin", "arch": "arm64"},
        {"os": "windows", "arch": "amd64"}
      ]
    }
  ],
  "warnings": null
}
EOF
```

```bash
# Create download endpoint for each platform
mkdir -p /var/www/registry/v1/providers/mycompany/myapp/1.0.0/download

cat > /var/www/registry/v1/providers/mycompany/myapp/1.0.0/download/linux/amd64 << 'EOF'
{
  "protocols": ["5.0"],
  "os": "linux",
  "arch": "amd64",
  "filename": "terraform-provider-myapp_1.0.0_linux_amd64.zip",
  "download_url": "https://registry.internal.company.com/files/terraform-provider-myapp_1.0.0_linux_amd64.zip",
  "shasums_url": "https://registry.internal.company.com/files/terraform-provider-myapp_1.0.0_SHA256SUMS",
  "shasums_signature_url": "https://registry.internal.company.com/files/terraform-provider-myapp_1.0.0_SHA256SUMS.sig",
  "shasum": "abc123def456...",
  "signing_keys": {
    "gpg_public_keys": [
      {
        "key_id": "MYCOMPANY_KEY_ID",
        "ascii_armor": "-----BEGIN PGP PUBLIC KEY BLOCK-----\n..."
      }
    ]
  }
}
EOF
```

## nginx Configuration

```nginx
# /etc/nginx/sites-enabled/opentofu-registry
server {
    listen 443 ssl http2;
    server_name registry.internal.company.com;

    ssl_certificate     /etc/ssl/certs/registry.crt;
    ssl_certificate_key /etc/ssl/private/registry.key;

    root /var/www/registry;

    # Service discovery
    location /.well-known/terraform.json {
        add_header Content-Type "application/json";
    }

    # Provider API - serve without extension
    location ~* ^/v1/providers/ {
        add_header Content-Type "application/json";
        # Try file, then with no extension, then 404
        try_files $uri $uri/ =404;
    }

    # Provider binary files
    location /files/ {
        root /var/www/registry;
        add_header Content-Type "application/zip";
    }

    # CORS for tooling
    add_header Access-Control-Allow-Origin "*";
    add_header Access-Control-Allow-Methods "GET";
}
```

## Building a Custom Provider for Internal Registry

```go
// main.go - Minimal provider structure
package main

import (
    "github.com/hashicorp/terraform-plugin-sdk/v2/plugin"
    "github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema"
)

func main() {
    plugin.Serve(&plugin.ServeOpts{
        ProviderFunc: Provider,
    })
}

func Provider() *schema.Provider {
    return &schema.Provider{
        Schema: map[string]*schema.Schema{
            "api_url": {
                Type:     schema.TypeString,
                Required: true,
            },
        },
        ResourcesMap: map[string]*schema.Resource{
            "myapp_service": resourceService(),
        },
    }
}
```

```makefile
# Makefile for building and publishing provider

VERSION ?= 1.0.0
BINARY = terraform-provider-myapp

build-all:
    for OS in linux darwin windows; do \
        for ARCH in amd64 arm64; do \
            GOOS=$$OS GOARCH=$$ARCH go build -o dist/$(BINARY)_$(VERSION)_$${OS}_$${ARCH}; \
            cd dist && zip $(BINARY)_$(VERSION)_$${OS}_$${ARCH}.zip $(BINARY)_$(VERSION)_$${OS}_$${ARCH}; \
            cd ..; \
        done; \
    done

publish: build-all
    # Generate SHA256SUMS
    cd dist && sha256sum *.zip > $(BINARY)_$(VERSION)_SHA256SUMS
    # Sign the checksums
    gpg --detach-sign $(BINARY)_$(VERSION)_SHA256SUMS
    # Copy to registry files directory
    cp dist/*.zip dist/*.SHA256SUMS dist/*.sig /var/www/registry/files/
    # Update version metadata (automated in production)
    ./scripts/update-registry-metadata.sh $(VERSION)
```

## Using GitLab as a Provider Registry

```bash
# GitLab supports Terraform provider packages natively

# Package the provider
cd dist/
curl --header "PRIVATE-TOKEN: $GITLAB_TOKEN" \
     --upload-file terraform-provider-myapp_1.0.0_linux_amd64.zip \
     "https://gitlab.company.com/api/v4/projects/$PROJECT_ID/packages/terraform/providers/mycompany/myapp/1.0.0/linux_amd64/file"

# GitLab auto-generates the registry metadata
```

```hcl
# Reference GitLab provider registry
terraform {
  required_providers {
    myapp = {
      source  = "gitlab.company.com/mycompany/myapp"
      version = "~> 1.0"
    }
  }
}
```

## Using the Internal Registry

```hcl
# In your OpenTofu configuration
terraform {
  required_providers {
    myapp = {
      source  = "registry.internal.company.com/mycompany/myapp"
      version = "~> 1.0"
    }
  }
}

provider "myapp" {
  api_url = "https://app.internal.company.com/api"
}
```

```hcl
# ~/.terraform.rc - Configure the registry
credentials "registry.internal.company.com" {
  token = "your-registry-token"
}
```

## Conclusion

An internal OpenTofu provider registry requires implementing the Terraform Registry Protocol's HTTP API: a service discovery endpoint and version/download endpoints for each provider. The simplest implementation uses nginx serving static JSON files, updated by scripts when new provider versions are published. GitLab's built-in Terraform registry is a managed alternative that handles the protocol automatically when you upload provider packages via the Package Registry API.
