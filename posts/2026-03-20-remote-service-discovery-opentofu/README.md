# How OpenTofu Remote Service Discovery Works

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: OpenTofu, Service Discovery, Remote Backend, Protocol, Internals, Infrastructure as Code

Description: Learn how OpenTofu discovers remote services - the .well-known/terraform.json protocol that enables registry API discovery, remote backend connectivity, and authentication service endpoints.

## Introduction

OpenTofu uses a service discovery protocol to find the API endpoints for registries, remote backends, and authentication services. When you interact with `registry.opentofu.org` or a private registry, OpenTofu first fetches `/.well-known/terraform.json` to discover what services are available and where their APIs live.

## The Service Discovery Protocol

```bash
# OpenTofu fetches .well-known/terraform.json from any hostname it interacts with

curl -s https://registry.opentofu.org/.well-known/terraform.json | jq .
```

Response:

```json
{
  "modules.v1": "/v1/modules/",
  "providers.v1": "/v1/providers/",
  "login.v1": "/oauth2/authorization",
  "state.v2": "/api/v2/",
  "tfe.v2.1": "/api/v2/"
}
```

Each key is a service identifier with version; the value is the base path for that service's API.

## Service Identifiers

| Service Key | Purpose |
|-------------|---------|
| `modules.v1` | Module registry API |
| `providers.v1` | Provider registry API |
| `login.v1` | OAuth 2.0 authentication |
| `state.v2` | Remote state storage (Terraform Cloud compatible) |
| `tfe.v2.1` | Terraform Enterprise API |

## How tofu init Uses Service Discovery

```bash
# Trace the discovery process
TF_LOG=DEBUG tofu init 2>&1 | grep "well-known\|discovery\|service"

# Output shows:
# DEBUG request to https://registry.opentofu.org/.well-known/terraform.json
# DEBUG discovered modules.v1 endpoint: /v1/modules/
# DEBUG discovered providers.v1 endpoint: /v1/providers/
```

## Setting Up a Private Registry with Service Discovery

Your private registry needs to serve the discovery endpoint:

```nginx
# nginx configuration for private OpenTofu registry
server {
    listen 443 ssl;
    server_name registry.mycompany.com;

    # Service discovery endpoint
    location /.well-known/terraform.json {
        add_header Content-Type application/json;
        return 200 '{
            "modules.v1": "/v1/modules/",
            "providers.v1": "/v1/providers/",
            "login.v1": "/oauth2/authorization"
        }';
    }

    # Module API (forward to backend)
    location /v1/modules/ {
        proxy_pass http://module-registry-backend:8080/v1/modules/;
    }

    # Provider API (forward to backend or mirror)
    location /v1/providers/ {
        proxy_pass http://provider-mirror-backend:8080/v1/providers/;
    }
}
```

## Authentication Discovery

When `login.v1` is present, `tofu login` uses it for OAuth:

```bash
# tofu login uses the login.v1 endpoint for OAuth
tofu login registry.mycompany.com

# OpenTofu fetches:
# https://registry.mycompany.com/.well-known/terraform.json
# Finds: "login.v1": "/oauth2/authorization"
# Redirects to: https://registry.mycompany.com/oauth2/authorization?...
```

Manual token configuration (bypass OAuth):

```hcl
# ~/.terraformrc
credentials "registry.mycompany.com" {
  token = "my-registry-token"
}
```

## Remote Backend Service Discovery

For Terraform Cloud/Enterprise compatible backends:

```hcl
# backend.tf
terraform {
  backend "remote" {
    hostname     = "app.terraform.io"   # Discovery fetched from here
    organization = "my-org"

    workspaces {
      name = "production-vpc"
    }
  }
}
```

OpenTofu discovers the `tfe.v2.1` endpoint and uses it for:
- Uploading configuration
- Streaming plan/apply logs
- Storing state

## Custom Service Discovery for Air-Gapped Environments

```python
#!/usr/bin/env python3
# internal-registry/discovery.py - minimal service discovery server

from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/.well-known/terraform.json')
def discovery():
    return jsonify({
        "modules.v1": "/v1/modules/",
        "providers.v1": "/v1/providers/"
        # No login.v1 - use static token auth instead
    })

@app.route('/v1/modules/<namespace>/<name>/<provider>/versions')
def module_versions(namespace, name, provider):
    # Return available module versions
    return jsonify({
        "modules": [{
            "versions": [
                {"version": "1.0.0"},
                {"version": "1.1.0"}
            ]
        }]
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, ssl_context='adhoc')
```

## Verifying Service Discovery

```bash
# Test that your registry returns valid service discovery
curl -s https://registry.mycompany.com/.well-known/terraform.json | \
  jq 'if has("modules.v1") then "✅ modules.v1 found" else "❌ modules.v1 missing" end'

# Test module listing works
curl -s "https://registry.mycompany.com/v1/modules/my-org/vpc/aws/versions" | \
  jq '.modules[0].versions | length'
```

## Conclusion

OpenTofu's service discovery protocol uses `.well-known/terraform.json` to find API endpoints for any hostname it interacts with. This enables a single hostname to serve multiple services (modules, providers, authentication, remote state) with each service independently discoverable. When setting up a private registry, implement the service discovery endpoint first - it's the entry point that OpenTofu uses to locate all other APIs. For air-gapped environments, a minimal HTTP server serving the discovery JSON and proxying requests to local package stores is sufficient.
